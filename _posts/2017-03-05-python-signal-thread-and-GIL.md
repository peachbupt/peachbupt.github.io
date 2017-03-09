# Python signal, thread and GIL

公司的gobuild是一套功能强大的分布式的构建和编译系统。这套系统会在编译之前根据代码库中的profile文件解析并下载构建模块所依赖头文件和动态库（所依赖库的版本和类型也可以在编译时动态指定)。这种松散的耦合关系非常高效的减轻了产品开发过程中各个项目组之间的依赖关系。

gobuild客户端的代码主要是基于python实现，当利用Make或者Scons构建的时候，调用gobuild的代码完成上述功能。然而在做local build的时候，经常在切换网络的时候碰到gobuild程序hang住，使用ctrl+C无法使进程退出的情况。本文即试图从python实现的角度讲解这种情况出现的原因。

本文讲忽略gobuild具体的实现过程，简化的问题如下：
	
	* based on CPython 2.7.*
	* Python thread1 is blocking in some IO function
	* Python main thead blocking in thread1.join()
	* why Python process can't handle signal ?

## Python 虚拟机

Python是一门动态的解释语言，支持过程式、面向对象、函数式、泛型等多种编程范式。CPython是指Python的interpret(相当于javac和java）是由C语言实现，CPython是python的参考实现，其他的Python实现方式还是PyPy,Jython等。

Python执行过程中，Python interpret会首先扫描相关的*.py文件进行词法和语法分析，并建立语法树，然后由编译器生成可以执行的Python字节码，这些bytecode会和其他一些静态执行信息（譬如Python的各种命令空间LEGB）存储在PyCodeObject中。Python文件中的每一个Code Block都会生成一个PyCodeObject的instance，这些PyCodeObject会根据命名空间的关系组织成一棵树，Python虚拟机在执行的过程中，很大一部分的时间消耗就是按照LEGB的顺序在从这棵树中确定一个符号对应的对象是什么。Python会将这些PyCodeObject和编译出的字节码存储在\*.pyc文件中。

PyCodeObject最后会被Python虚拟机翻译成关的机器代码交给操作系统执行。Python虚拟机其实是一个叫做PyEval_EvalFrameEx的函数，这个函数可以简化为一个loop-switch的函数。它会遍历bytecode，并根据[Python Bytecode Instruction](!https://docs.python.org/3/library/dis.html#opcode)的类型调用相应的C函数实现，从而实现各种相应的功能。与X86机器上可执行文件执行的时候会构建运行时栈帧类似，Python虚拟机在执行的时候也需要一个PyFrameObject来存储运行时栈信息以及局部变量、支持闭包的变量等信息。每个PyFrameObject都会mapping到一个PyCodeObject，Python虚拟机执行时会根据将PyCodeObject创建PyFrameObject，这些PyFrameObject也会在调用相应的Python Bytecode Instruction时进行压栈和出栈的操作。

当然，Python是基于C语言实现的，因此Python进程的内存中也会存在一个标准的C栈帧，ESP和EBP寄存器同样会在C函数调用的时候进行更新。Python维护的这个PyFrameObject的“栈帧”只是用来实现Python语言相关的操作，譬如命名空间，闭包，和Python虚拟机执行需要的信息。所以简单理解，python是一个包含了编译器和虚拟机的c语言程序，编译器将符合python语法的文件翻译成AST bytecode，虚拟机解析bytecode并根据Python的指令集调用相应的C语言函数。

## Python Thread & GIL

Python中的线程就是操作系统原生的线程。当调用Python的Thread或者Threading创建线程的时候，Python会直接创建一个操作系统原生的线程和一个PyThreadState的结构体,然后在新的线程内调用Python虚拟机函数PyEval_EvalFrameEx。这个PyThreadState会包含当前线程的PyFrameObject、Thread Id以及一些per-thread的执行信息。显然，在多个thread中同时执行多个虚拟机会给加大Python Interpert实现和Python的内存管理以及回收的难度，因此Python的Interpret在同一时刻只允许一个线程执行Python的虚拟机函数，具体的实现方式是在Python虚拟机中透过一个叫做GIL的锁串行化Python Bytecode的解析过程。Python的虚拟机会维护一个PyThreadState的链表并在线程调度的时候将一个叫\_PyThreadState\_Current的指针指向各个Thread中的PyThreadState，从而完成在各个Python Thread间轮流执行。

![MacDown logo](/images/posts/2017-03-02/threads.png)

既然Python的Thread如此鸡肋，那么Python为什么还要引入Thread呢? 一个主要的原因是为了更好的支持I/O操作。Python Thread在执行阻塞的IO操作时会释放GIL锁，这样可以创建多个线程分别用来执行IO密集型和CPU密集型的操作。

GIL在windows上面是利用Event实现，在Linux上面利用Mutex和Condition来实现。当一个拥有GIL的Thread在执行的时候，其他的Thread会被Block在WaitForSingleObject或者Condition wait(),当正在执行的Thread释放GIL锁，操作系统会从等待的线程中选取一个继续执行，这种的wait会产生大量的系统调用，而这也是Python在多核系统上面系能急剧下降的原因。

Python Thread同样使用操作系统的线程调度机制，Python Thread会在两种情况下触发线程调度机制，第一种就是上面所说的阻塞IO和调用time.sleep()的时候; 第二种是Python虚拟机会每隔100个ticks主动释放一次GIL锁，从而触发操作系统的线程调度机制。Python虚拟机会在PyEval_EvalFrameEx执行过程中递减ticks，当ticks == 0的时候触发一个check()函数，在check()函数中，Python虚拟机会停止执行bytecode并保存线程信息，然后释放GIL，这个时候如果操作进行线程调度，其他的Python线程会获得GIL并开始执行，否则当前线程会重新得到GIL锁并继续执行。

```c++
        if (--_Py_Ticker < 0) {
            _Py_Ticker = _Py_CheckInterval;
            tstate->tick_counter++;
            if (pendingcalls_to_do) {
                if (Py_MakePendingCalls() < 0) {
                    why = WHY_EXCEPTION;
                    goto on_error;
                }
                if (pendingcalls_to_do)
                    _Py_Ticker = 0;
            }
#ifdef WITH_THREAD
            if (interpreter_lock) {
                /* Give another thread a chance */
                if (PyThreadState_Swap(NULL) != tstate)
                    Py_FatalError("ceval: tstate mix-up");
                PyThread_release_lock(interpreter_lock);

                /* Other threads may run now */
                PyThread_acquire_lock(interpreter_lock, 1);

                if (PyThreadState_Swap(tstate) != NULL)
                    Py_FatalError("ceval: orphan tstate");

                /* Check for thread interrupts */
                if (tstate->async_exc != NULL) {
                    x = tstate->async_exc;
                    tstate->async_exc = NULL;
                    PyErr_SetNone(x);
                    Py_DECREF(x);
                    why = WHY_EXCEPTION;
                    goto on_error;
                }
            }
#endif
        }
```

然而，Python虚拟机对ticks的定义却稍显混乱。一个tick既可以对应到一个bytecode operation，也可以对应到一句python语句，更有甚者，你可以写一段python代码，这段代码会一直执行并且在执行过程中不会递减ticks，这样子的python代码被称为uninterruptible操作，譬如thread.join()就是一个uninterruptible的操作。

![MacDown logo](/images/posts/2017-03-02/ticks.png)

## Signal handling messiness

CPython的Interpret只能够在Main Thread中处理signal操作。而在问题发生时，主线程正调用一个uninterruptiable的thread.join()函数，如下属的代码所示，thread.join()在IO线程结束之前不会退出，而IO线程却因为网络问题等待TCP的retransmit超时，TCP的retransmit超时一般[需要13到30min](!http://stackoverflow.com/questions/13085676/tcp-socket-no-connection-timeout),所以Python不会及时的相应CTRL+C的signal信号。

一个解法是在主线程中调用Thread.join()的时候添加timeout的参数。

```python
  def join(self, timeout=None):
        self.__block.acquire()
        try:
            if timeout is None:
                while not self.__stopped:
                    self.__block.wait()
                if __debug__:
                    self._note("%s.join(): thread stopped", self)
            else:
                deadline = _time() + timeout
                while not self.__stopped:
                    delay = deadline - _time()
                    if delay <= 0:
                        if __debug__:
                            self._note("%s.join(): timed out", self)
                        break
                    self.__block.wait(delay)
                else:
                    if __debug__:
                        self._note("%s.join(): thread stopped", self)
        finally:
            self.__block.release()

```

## Extend: GIL and Multiple Core

Python的单线程进行机制会跟操作系统在多核CPU上面的线程调度产生冲突，操作系统会倾向于将不同的线程affinity到不同的CPU Core从而做到并发，而Python的GIL却限制在同一时刻只允许一个线程执行，因此在多核上面没有获得GIL锁的线程会产生的大量的系统调用，从而导致Python的性能下降很多。

![MacDown logo](/images/posts/2017-03-02/mcores.png)


##Reference
1. 《Python 源码剖析》
2. http://www.dabeaz.com/python/UnderstandingGIL.pdf



