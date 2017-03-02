# MongoDB

[TechBoard](https://github.com/peachbupt/TechBoard)最终选用了MongoDB存储爬虫爬到的网站信息。当然对于[TechBoard](https://github.com/peachbupt/TechBoard)这种自娱自乐的项目，任何一个Document-Oriented的数据库甚至直接将数据序列化存储在文件中都能够满足需求。不过现在正值春节假期，正适合利用零零碎碎的时间从scalability, availability,以及performance的角度学习一些数据库的设计基本准则。

## CAP and DataBase

CAP是分布式系统的基本原则，CAP理论断言任何基于网络的数据共享系统，最多只能满足数据一致性、可用性、分区容忍性三要素中的两个要素。

理解CAP理论的最简单方式是想象两个节点分处分区两侧。允许至少一个节点更新状态会导致数据不一致，即丧失了C性质。如果为了保证数据一致性，将分区一侧的节点设置为不可用，那么又丧失了A性质。除非两个节点可以互相通信，才能既保证C又保证A，这又会导致丧失P性质。一般来说跨区域的系统，设计师无法舍弃P性质，那么就只能在数据一致性和可用性上做一个艰难选择。不确切地说，NoSQL运动的主题其实是创造各种可用性优先、数据一致性其次的方案；而传统数据库坚守ACID特性（原子性、一致性、隔离性、持久性），做的是相反的事情。^[2]

![DataBase and CAP](../images/posts/2017-03-02/database_and_cap.png)

## ACID and BASE

ACID和BASE是数据库事务设计的两种哲学。ACID强调的是保证数据库事务的一致性，而BASE侧重的是数据库的可用性。对于诸如MySQL，Oracle这些传统的关系型数据库，需要保证的是数据的Consistency，因此事务的ACID是充要条件。而对于像Dynamo这些应用于website的非关系型数据库，需要保证的是非常高的Availablity，而对于数据的Consistency却没有强的要求，因此BASE中软状态和最终一致性这两种技巧更适合这种强调高可用性的需求。

## SQL & NOSQL

SQL是一种结构化查询语言(Structured Query Language)，NOSQL是对不同于传统的关系型数据库的数据库管理系统的统称。NoSQL用于超大规模数据的存储，这些类型的数据存储不需要固定的模式，无需多余操作就可以横向扩展。

NOSQL按照存储的的类型或组织形式可以细分为key-value，document，列，xml，图等形式。不过我目前只用过key-value的memcached和基于Document的mongodb.

## MongoDB Basic

MongoDB 将数据存储为一个文档，数据结构由键值(key=>value)对组成。MongoDB文档类似于 JSON对象。字段值可以包含其他文档，数组及文档数组。

| SQL 术语/概念  | Mongo DB 术语/概念  | 解释/说明 |
|:-------------|:---------------:|:-------------|
| database      | database 		 |         数据库 |
| Table      	   | collection      | 数据库表/集合   |
| Row 				| Document        |  数据记录行/文档 |
| Column 				| Field        |  数据字段/域 |
| Index 				| index        |  索引 |
| Table joins 				|        |  MongoDB 不支持 |
| primary key 				|  primary key      |  主键，MongoDB自动将ID设为主键 |

一个mongo db中可以包含多个数据库，使用_show dbs_可以列出所有的数据库,

~~~
> show dbs
admin  0.000GB
board  0.000GB
local  0.000GB
study  0.000GB
test   0.000GB
~~~
使用_use_命令切换到某一个数据库，当数据库没有的时候会创建一个新的database,使用_dp.dropDatabase()_和_db.collection.drop()_可以删除数据库和集合。

~~~
> use board
switched to db board
~~~

MongoDB中的文档是一个键值(key-value)对。MongoDB 的文档不需要设置相同的字段，并且相同的字段不需要相同的数据类型，这与关系型数据库有很大的区别。使用insert可以向collection中插入文档.

~~~
> db.study.insert({"name":"alice", "location":"Beijing"})
WriteResult({ "nInserted" : 1 })
~~~

可以使用_update_方法来更新db collection中的文档的内容。

~~~
> db.study.update({'name':'alice'},{$set:{'name':'BoB'}})
WriteResult({ "nMatched" : 1, "nUpserted" : 0, "nModified" : 1 })
~~~

使用_remove__删除collection的文档。

~~~
> db.study.remove({"name":"BoB"})
WriteResult({ "nRemoved" : 1 })
~~~

mongodb 查询数据的语法格式如下.

~~~
> db.COLLECTION_NAME.find()
> db.COLLECTION_NAME.find().pretty()
> db.COLLECTION_NAME.find({$key:$value})
~~~

## PyMongo vs MongoEngine

Pymongo和MongoEngine是Python中操作MongoDB的两个库。Pymongo是利用Python语言封装了上述MongoDB的API，并以JSON格式的数据作为输入和输出。而MongoEngine是介于DataBase和Model之间的ORM层，它定义了Model层操作数据库Document的一些固定schema，因此可以通过访问类成员变量的方式访问数据库collection中的Document数据。

##reference
[1] https://www.quora.com/What-is-the-relation-between-SQL-NoSQL-the-CAP-theorem-and-ACID

[2] http://www.infoq.com/cn/articles/cap-twelve-years-later-how-the-rules-have-changed/

[3] http://blog.nahurst.com/visual-guide-to-nosql-systems

[4] https://www.quora.com/Why-cant-a-distributed-NoSQL-database-support-all-CAP-rules

[5] http://www.runoob.com/mongodb/nosql.html

[6] http://stackoverflow.com/questions/5712857/pymongo-vs-mongoengine-for-django
