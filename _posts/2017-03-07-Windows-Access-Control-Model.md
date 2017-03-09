## Windows Access Control

最近这一周在写一个嵌入到IE浏览器的BHO跟其他进程做IPC的代码，因此花费很多时间学习了下Windows操作系统的Access Control机制以及浏览器的Protect Mode和Enhanced Protect Mode。总体而言，Windows的Access Control机制可谓架构清楚合理，而IE浏览器在兼容32bit/64bit,保证安全性和用户体验之间的改变与折中也充分体现了一个项目开发过程中的创新和妥协。

本篇文章会重点谈下Windows Access Control的基本Model，以及后期引入的Client Impersonation, MIC，AppContainer，UAC等authrization机制，下篇会结合这些机制来谈下IE在不同windows版本中间的security evolution.

”Talk is cheap，show me the code“，本文将简化论述，在每个小节中用代码来做论据。

### Access Control Model


![](/images/posts/2017-03-07/windows_acl_model.png)

Windows Access Control基本的单元是**进程(线程)**和**securable object**。

当用户登录操作系统并完成认证之后，操作系统会读取本机或者AD的数据库，生成一个access token, 这个token包含了用户和用户组的信息以及用户和用户组拥有的privileges。运行在这个用户session中的每一个process都会从父进程继承一份access token的拷贝。

所谓的**securable object**是指包含**security descriptor**的对象。所有的windows有名对象(文件，mutex)以及一些无名对象(进程，线程)都属于**securable object**。当创建一个**securable object**，相关的API会给这些内核对象指定一个**security descriptor**。这些**security descriptor**用来表明object的所有者以及其他的用户和用户组是否可以访问这些object。

* windows access control的基本单元包含线程是因为线程支持Impersonation，详见下面的client/serve access control。
* CreateProcess的lpProcessAttributes参数就包含一个lpSecurityDescriptor属性，可以在CreateProcess的时候指定一个**security descriptor**，而在lpProcessAttributes == NULL的时候，新创建的进程会得到一个default的security descriptor)。大部分创建内核对象的API都可以直接继承默认的security descriptor。
* **security descriptor**用来表示object的owner，同时还可以包含如下信息.
	* **DACL, discretionary access control list**, 用来表示那些用户或者用户组可以访问object;
	* **SACL, system access control list**，用来标识操作系统如何audit这个object，那些access需要log到系统安全相关的event log中;
	* 每个ACL中包含的是一个**access control entries (ACEs)**的链表。每个ACE中包含一个SID和SID对应的访问权限。这个SID可以代指一个用户，一个用户组以及一个logon的session;
* **privilege**是指一个用户或者用户组是否能够执行操作系统关机，加载设备驱动，更改系统时间的权限，与ACL的机制不同，**privilege**是由管理员指定的.

### DACLs and ACEs

![](/images/posts/2017-03-07/process_and_dacl.png)

当一个**securable object**不包含任何**DACL**的时候，操作系统允许所有用户访问这个object; 用一个object包含**DACL**的时候，操作系统只允许在DACL中**access control entries (ACEs)**允许的用户或者用户组访问这个object;当一个DACL中即包含access denied又包含access allowed的规则时，access denied的rule需要放在list的起始。

**The security descriptor definition language (SDDL)**定义了ACE中安全描述符的格式，可以使用**ConvertSecurityDescriptorToStringSecurityDescriptor** 或者**ConvertStringSecurityDescriptorToSecurityDescriptor**将一个String转换为安全描述符。

### Access Control Demo

Windows系统的security properties列举出了一个文件的所有安全描述符。下面的demo从进程中获取access token，然后从access token得到相关的进程所属于的用户和用户组以及操作系统允许的privilege. 同理这个demo的另外一部分会读取某个文件的security descriptor并检测当前用户是否有权限write这个文件.

https://github.com/peachbupt/Lab/tree/master/WindowsACLDemo

![](/images/posts/2017-03-07/acl_model_demo.png)

### Client/Server Access Control

Clinet/Server Access Control是指如何在一个进程/线程中用另外一个SID去访问系统的资源和创建对象。一个受保护的server(这里的Server可以是不同主机也可以是同一主机上面的IPC Server)会有一个标识自己和权限的SID，当这个server接受到client的请求时，有两种方式去以client的SID去访问系统资源.

* 第一种是create一个thread **impersonate**客户端。这个thread会产生一个impersonate access token，并用这个token完成ACL或者权限的检查。
	* Windows 中有很多API可以用来做impersonate，实际上正在维护的软件有的很多进程是跑在SYSTEM权限下，因此经常要impersonate用户的session去做一些事情。常用的API有IPC相关的(**DdeImpersonateClient**)和impersonate已经logon的用户的**ImpersonateLoggedOnUser**。
* 第二种是server可以获得client的用户名和密码，然后使用client的credential去登录系统从而获得access token.
	* 调用LogonUser的进程必须有**SE_TCB_NAME**的权限.

### Client/Server Access demo

这个demo会在SYSTEM用户目录下以当前的登录用户打开一个技术本。具体做法为，在**PSEXEC**中以SYSTEM权限执行demo程序，demo程序会在用户session中查找explorer并复制用户的access token，然后调用**CreateProcessWithUser**在用户的session中impersonate一个记事本进程。
https://github.com/peachbupt/Lab/tree/master/WindowsACLDemo

![](/images/posts/2017-03-07/demo.png)

### Access Control for Application Resources

MSDN上的这一段看得我莫名其妙，看上去是在讲windows server 2003上面的Service isolation,即每一个Service可以创建一个SID用来保护自己创建的资源不被同样跑在SYSTEM权限下的其他Service访问,这一段就不写demo了.

### Mandatory Integrity Control

**Mandatory Integrity Control (MIC)**是Windows Vista引入的一种安全机制。其主要思想是在ACL Model基础上(同样的内核对象)添加了基于**Integrity Levels (IL)-**的保护机制，这种IL用来细分同一用户权限下不同信任等级进程的访问权限。

具体来说，Windows一共定义了四个等级的Integrity Level SIDs,分别为Low (SID: S-1-16-4096), Medium (SID: S-1-16-8192), High (SID: S-1-16-12288), and System (SID: S-1-16-16384), 这四个SID会在创建access token时被分配到access token group中。同样操作系统在创建object的时候也会在security descriptor中添加一个mandatory label的ACE。Windows系统会在进行ACL检查之前首先对IL进行匹配，基本原则是低IL的进程不能与高IL的object进行interact，这里的interact指文件访问，IPC以及诸如使用**CreateRemoteThread()**进行DLL injection，使用**WriteProcessMemory()**进行跨进程写数据的操作。

当然上面所说的**LI**的进程不能与**HI**的object交互有点太绝对，譬如windows的HKLM注册表允许HIGH Level的用户有full control，但是对于低Level的用户仍然保留读权限。windows操作系统是通过在access token中添加**mandatory policies**和在security descriptor中添加**Mandatory label policies**来实现上述的控制。譬如在object的Mandatory label polices中添加SYSTEM_MANDATORY_POLICY_NO_READ_UP可以禁止低level的进程访问本object。

**Mandatory Integrity Control**被apply到诸如service，com，registry hive等很多组件并被用来实现了UAC和IE的protected mode。**Integrity Control**的实现机制决定了他与ACL的SIDs以及privilege有非常紧密的耦合关系。譬如在windows services中只有LocalSystem, LocalService, and NetworkService三个SIDs拥有**SE\_IMPERSONATE\_NAME** 权限，而windows的integrity机制将**SE\_IMPERSONATE\_NAME**的权限与SYSTEM Integrity Level想关联，因此低IL的进程不能够impersonate出高IL的client。

对于com或者其他IPC,禁止低IL的进程与高IL的对象建立连接太过于严格，因此可以通过在创建com或者IPC对象的的时候指定特殊的SDDL的方式降低这些对象的IL，从而允许低IL的进程与高IL进程中的对象进行IPC或者com bind操作，具体方法如下.

* Define a mandatory label with a NO_EXECUTE_UP (NX) policy for low integrity level. The SDDL for the object security descriptor with the mandatory label policy for low integrity is the following:

		O:BAG:BAD:(A;;0xb;;;WD)S:(ML;;NX;;;LW)

* Convert the string security descriptor to a binary security descriptor.
* Set the Launch Permissions for the CLSID for the server CLSID using the binary security descriptor.

### Mandatory Integrity Control demo

Demo中的程序会读取当前用户的Integrity Level, 另外一个函数在High Level的进程中以**SID: S-1-16-4096**创建一个low IL level运行的notepad.
https://github.com/peachbupt/Lab/tree/master/WindowsACLDemo

![](/images/posts/2017-03-07/create_low_IL_process.png)



























