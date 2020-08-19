基于原书第二版。  

- [第5章 索引与算法（待完善）](#第5章-索引与算法待完善)
  - [introduce](#introduce)
  - [概述](#概述)
  - [B+树索引的管理](#b树索引的管理)
    - [Fast Index Creation](#fast-index-creation)
    - [Online Schema Change](#online-schema-change)
    - [Online DDL](#online-ddl)
  - [全文检索](#全文检索)
    - [倒排索引](#倒排索引)
    - [InnoDB全文检索](#innodb全文检索)
    - [全文检索](#全文检索-1)
      - [NATURAL LANGUAGE](#natural-language)
      - [Boolean](#boolean)
      - [Query Expansion](#query-expansion)
- [第6章 锁](#第6章-锁)
  - [lock与latch](#lock与latch)
  - [锁](#锁)
    - [类型](#类型)
    - [一致性非锁定读 与 MVCC](#一致性非锁定读-与-mvcc)
    - [一致性锁定读](#一致性锁定读)
    - [自增长与锁](#自增长与锁)
    - [外键和锁](#外键和锁)
  - [锁的算法](#锁的算法)
    - [幻读 aka Phantom Problem](#幻读-aka-phantom-problem)
    - [关于阻塞和死锁](#关于阻塞和死锁)
  - [锁升级](#锁升级)
  - [mysql的事务隔离级别](#mysql的事务隔离级别)


# 第5章 索引与算法（待完善）  

## introduce
说明：看目录感觉和《MySQL技术内幕：SQL编程》中的 第九章 索引内容差不多，相近部分结合[MySQL技术内幕：SQL编程_读书笔记](/mysql/MySQL技术内幕：SQL编程_读书笔记.md#introduce)查看。此处只补充一些不同的部分。  

## 概述  

InnoDB存储引擎支持以下几种常见的索引：B+树索引、全文索引、哈希索引。  

在InnoDB存储引擎中，哈希索引是自适应的，InnoDB存储引擎会根据表的使用情况自动为表生成哈希索引，不能人为干预是否在一张表中生成哈希索引。  

B+树索引并不能找到一个给定键值的具体行。B+树索引能找到的只是被查找数据行所在的页。然后数据库通过把页读入到内存，再在内存中进行查找，最后得到要查找的数据。  

## B+树索引的管理  
### Fast Index Creation  
快速索引创建。FIC方式只限定于辅助索引，对于主键的创建和删除同样需要重建一张表。  
对于辅助索引的创建，InnoDB存储引擎会对创建索引的表加上一个S锁（在创建的过程中只能对该表进行读操作）。在创建的过程中，不需要重建表，因此速度较之前提高很多，并且数据库的可用性也得到了提高。  
删除辅助索引操作就更简单了，InnoDB存储引擎只需更新内部视图，并将辅助索引的空间标记为可用，同时删除MySQL数据库内部视图上对该表的索引定义即可。  

### Online Schema Change  
Online Schema Change是一个php脚本。  
首先介绍InnoDB存储引擎中对数据表的DDL操作（改表结构、建索引等）流程：1 创建一张新表，2 将旧表数据导入新表，3 删除原表，4 把临时表重命名为原来的表名。用户对于一张大表进行索引的添加和删除操作，那么这会需要很长的时间，同时表不可访问。  
前面的FIC是一种提高索引创建速度的机制。这部分的OSC是为实现修改表结构时依然可以访问原表。  

OSC的大致流程：  
1. 验证原表是否符合OSC条件。  
2. 创建一张和原表一样的新表。  
3. 对新表进行ALTER TABLE操作。  
4. 创建deltas表，该表记录所有对原表的DML操作。  
5. 对原表创建INSERT、UPDATE、DELETE操作的触发器。触发操作产生的记录被写入到deltas表。  
6. 开始OSC操作的事务。  
7. 将原表中的数据写入到外部文件。为了减少对原表的锁定时间，这里通过分片（chunked）将数据输出到多个外部文件，之后步骤会将外部文件的数据导入到copy表中。分片的大小可以指定，默认值是500000。  
8. 在导入到新表前，删除新表中所有的辅助索引。  
9. 将导出的分片文件导入到新表。  
10. 将OSC过程中原表DML操作的记录应用到新表中，这些记录被保存在deltas表中。  
11. 重新创建辅助索引。  
12. 再次进行DML日志的回放操作，这些日志是在上述创建辅助索引中过程中新产生的日志。  
13. 将原表和新表交换名字。整个操作需要锁定2张表，不允许新的数据产生。改名是一个很快的操作，阻塞的时间非常短。  

### Online DDL  
MySQL 5.6版本开始支持Online DDL（在线数据定义）操作，其允许辅助索引创建的同时，还允许其他诸如INSERT、UPDATE、DELETE这类DML操作，这极大地提高了MySQL数据库在生产环境中的可用性。  

InnoDB存储引擎实现Online DDL的原理是在执行创建或者删除操作的同时，将INSERT、UPDATE、DELETE这类DML操作日志写入到一个缓存中。待完成索引创建后再将重做应用到表上，以此达到数据的一致性。  

## 全文检索  
全文检索（Full-Text Search）是将存储于数据库中的整本书或整篇文章中的任意内容信息查找出来的技术。它可以根据需要获得全文中有关章、节、段、句、词等信息，也可以进行各种统计和分析。  

### 倒排索引  
全文检索通常使用倒排索引（inverted index）来实现。倒排索引同B+树索引一样，也是一种索引结构。它在辅助表（auxiliary table）中存储了单词与单词自身在一个或多个文档中所在位置之间的映射。这通常利用关联数组实现，其拥有两种表现形式：inverted file index，其表现形式为{单词，单词所在文档的ID}；full inverted index，其表现形式为{单词，（单词所在文档的ID，在具体文档中的位置）}。  

### InnoDB全文检索  
InnoDB存储引擎从1.2.x版本开始支持全文检索的技术，其采用fullinverted index的方式。在全文检索的辅助表中，有两个列，一个是word字段，另一个是ilist字段（DocumentId，Position），并且在word字段上有设有索引。  

辅助表（Auxiliary Table）是持久的表，存放于磁盘上。然而在InnoDB存储引擎的全文索引中，还有另外一个重要的概念FTS Index Cache（全文检索索引缓存），其用来提高全文检索的性能。  

关于FTS Index Cache的原文描述是：  
FTS Index Cache是一个红黑树结构，其根据（word，ilist）进行排序。这意味着插入的数据已经更新了对应的表，但是对全文索引的更新可能在分词操作后还在FTS Index Cache中，Auxiliary Table可能还没有更新。InnoDB存储引擎会批量对Auxiliary Table进行更新，而不是每次插入后更新一次Auxiliary Table。当对全文检索进行查询时，Auxiliary Table首先会将在FTS Index Cache中对应的word字段合并到Auxiliary Table中，然后再进行查询。这种merge操作非常类似之前介绍的Insert Buffer的功能，不同的是Insert Buffer是一个持久的对象，并且其是B+树的结构。然而FTSIndex Cache的作用又和Insert Buffer是类似的，它提高了InnoDB存储引擎的性能，并且由于其根据红黑树排序后进行批量插入，其产生的AuxiliaryTable相对较小。  

概括一下原文内容：向表中插入数据后，对新插入的全文检索字段进行分词操作，分词操作的结果会先存储到FTS Index Cache，然后再通过批量更新写入到磁盘。当对全文检索进行查询时，Auxiliary Table首先会将在FTS Index Cache中对应的word字段合并到Auxiliary Table中，然后再进行查询。  

数据库宕机时一些FTS IndexCache中的数据库可能未被同步到磁盘上。那么下次重启数据库时，当用户对表进行全文检索（查询或者插入操作）时，InnoDB存储引擎会自动读取未完成的文档，然后进行分词操作，再将分词的结果放入到FTS IndexCache中。  

文档中分词的插入操作是在事务提交时完成，然而对于删除操作，其在事务提交时，不删除磁盘Auxiliary Table中的记录，而只是删除FTS CacheIndex中的记录，并记录其FTS Document ID，并将其保存在DELETED auxiliary table中。由于前述的删除机制，随着应用程序的使用，索引会变得非常大，即使索引中的有些数据已经被删除，查询也不会选择这类记录。为此，InnoDB存储引擎提供了一种方式，允许用户手工地将已经删除的记录从索引中彻底删除，该命令就是OPTIMIZE TABLE。因为OPTIMIZE TABLE还会进行一些其他的操作，如Cardinality的重新统计，若用户希望仅对倒排索引进行操作，那么可以通过参数innodb_optimize_fulltext_only进行设置。  

当前InnoDB存储引擎的全文检索还存在以下的限制：  
1. 每张表只能有一个全文检索的索引。  
2.  由多列组合而成的全文检索的索引列必须使用相同的字符集与排序规则。  
3. 不支持没有单词界定符（delimiter）的语言，如中文、日语、韩语等。  

### 全文检索  
语法为：  

    MATCH(col1,col2,...) AGAINST (expr [search_modifier])
    search_modifier:
        {
              IN NATURAL LANGUAGE MODE
            | IN NATURAL LANGUAGE MODE WITH QUERY EXPANSION
            | IN BOOLEAN MODE
            | WITH QUERY EXPANSION
        }

MATCH指定了需要被查询的列，AGAINST指定了使用何种方法去进行查询。  

#### NATURAL LANGUAGE  
查询fts_a表中body字段含有Porridge的记录：  

    SELECT * FROM fts_a
    WHERE MATCH(body)
    AGAINST ('Porridge' IN NATURAL LANGUAGE MODE);

在WHERE条件中使用MATCH函数，查询返回的结果是根据相关性（Relevance）进行降序排序的，即相关性最高的结果放在第一位。相关性的计算依据以下四个条件：1 word是否在文档中出现。2 word在文档中出现的次数。3 word在索引列中的数量。4 多少个文档包含该word。  

查看相关性(Relevance列即为记录与Porridge的相关性)：  

    SELECT fts_doc_id,body,
    MATCH(body) AGAINST ('Porridge' IN NATURAL LANGUAGE MODE)
    AS Relevance
    FROM fts_a;

#### Boolean  
MySQL数据库允许使用IN BOOLEAN MODE修饰符来进行全文检索。  
下面的SQL语句用于查询有字符串Pease但没有hot的文档：

    SELECT *FROM fts_a
    WHERE MATCH(body) AGAINST ('+Pease -hot' IN BOOLEAN MODE)\G;

更多参见原书。  

#### Query Expansion  
全文检索的扩展查询。该查询分为两个阶段。第一阶段：根据搜索的单词进行全文索引查询。第二阶段：根据第一阶段产生的分词再进行一次全文检索的查询。  
例：第一阶段搜索的词为MicroSoft，搜索结果为一条：MicroSoft Windows，第二阶段以MicroSoft、Windows为搜索词，返回包含这两个词的结果。  


# 第6章 锁  
数据库系统使用锁是为了支持对共享资源进行并发访问，提供数据的完整性和一致性。  
MyISAM引擎，其锁是表锁设计。并发情况下的读没有问题，但是并发插入时的性能要差一些。  

## lock与latch  
latch一般称为闩锁（轻量级的锁），因为其要求锁定的时间必须非常短。若持续的时间长，则应用的性能会非常差。在InnoDB存储引擎中，latch又可以分为mutex（互斥量）和rwlock（读写锁）。其目的是用来保证并发线程操作临界资源的正确性，并且通常没有死锁检测的机制。  
lock用来锁定的是数据库中的对象，如表、页、行。并且一般lock的对象仅在事务commit或rollback后进行释放（不同事务隔离级别释放的时间可能不同）。有死锁机制。lock在整个事务过程中持续。  

## 锁  
### 类型  
InnoDB实现了以下两种行级锁：共享锁（S Lock），允许事务读一行数据。排他锁（X Lock），允许事务删除或更新一行数据。  
InnoDB存储引擎支持多粒度（granular）锁定，这种锁定允许事务在行级上的锁和表级上的锁同时存在，支持意向锁（Intention Lock）。加锁时需要对更粗粒度的对象先加意向锁。  
两种意向锁：1）意向共享锁（IS Lock），事务想要获得一张表中某几行的共享锁。2）意向排他锁（IX Lock），事务想要获得一张表中某几行的排他锁。  
从InnoDB1.0开始，在INFORMATION_SCHEMA架构下添加了表INNODB_TRX、INNODB_LOCKS、INNODB_LOCK_WAITS。通过这三张表，用户可以更简单地监控当前事务并分析可能存在的锁问题。  

### 一致性非锁定读 与 MVCC  
一致性的非锁定读（consistent nonlocking read）是指InnoDB存储引擎通过行多版本控制（multi versioning）的方式来读取当前执行时间数据库中行的数据。如果读取的行正在执行DELETE或UPDATE操作，这时读取操作不会因此去等待行上锁的释放。相反地，InnoDB存储引擎会去读取行的一个快照数据。不需要等待访问的行上X锁的释放，快照数据是指该行的之前版本的数据，该实现是通过undo段来完成。而undo用来在事务中回滚数据，因此快照数据本身是没有额外的开销。  

快照数据其实就是当前行数据之前的历史版本，每行记录可能有多个版本。一个行记录可能有不止一个快照数据，一般称这种技术为行多版本技术。由此带来的并发控制，称之为多版本并发控制（MultiVersion Concurrency Control，MVCC）。  

在事务隔离级别READ COMMITTED和REPEATABLE READ（InnoDB存储引擎的默认事务隔离级别）下，InnoDB存储引擎使用非锁定的一致性读。然而，对于快照数据的定义却不相同。  

在READ COMMITTED事务隔离级别下，对于快照数据，非一致性读总是读取被锁定行的最新一份快照数据。  
在REPEATABLE READ事务隔离级别下，对于快照数据，非一致性读总是读取事务开始时的行数据版本。  

### 一致性锁定读  
InnoDB存储引擎对于SELECT语句支持两种一致性的锁定读（locking read）操作：1） SELECT…FOR UPDATE（行记录加一个X锁）；2） SELECT…LOCK IN SHARE MODE（对读取的行记录加一个S锁）。注意：SELECT…FOR UPDATE，SELECT…LOCK IN SHARE MODE必须在事务中，务必加上BEGIN，STARTTRANSACTION或者SET AUTOCOMMIT=0。  

### 自增长与锁  
复杂。。推荐看原书。以下概述：  
参数innodb_autoinc_lock_mode来控制自增长的模式，该参数的默认值为1。  
为0时，会通过AUTO-INC locking方式，在插入时会短暂的加表锁（会很快释放，不会像其他锁持续到事务结束）。但并发插入性能较差。  
为1时，对于simple insert，会用互斥量mutex堆内存中的计数器进行累加的操作，如果不会滚的话自增的列是连续的。对于其他的同为0时相同。  
为2时，对于insert like（所有插入语句），都会用互斥量。并发性能好，但自增长的值可能不连续（对于同一个语句来说它所得到的auto_incremant值可能不是连续的。参照下面notice），而且对于statement-base replication会出现问题，应使用row-base replication。  
notice：两个会话中分别插入ab，cd，那么既有可能是（1,a）,(2,b)，（3，c），（4，d）也有可能是（3，b），（2，c）。  

### 外键和锁  
在InnoDB存储引擎中，对于一个外键列，如果没有显式地对这个列加索引，InnoDB存储引擎自动对其加一个索引，因为这样可以避免表锁。  
对于外键值的插入或更新，首先需要查询父表中的记录，即SELECT父表。但是对于父表的SELECT操作，不是使用一致性非锁定读的方式，因为这样会发生数据不一致的问题，因此这时使用的是SELECT…LOCK IN SHARE MODE方式，即主动对父表加一个S锁。  

## 锁的算法  
InnoDB存储引擎有3种行锁的算法，其分别是：  
Record Lock：单个行记录上的锁  
Gap Lock：间隙锁，锁定一个范围，但不包含记录本身  
Next-Key Lock∶Gap Lock+Record Lock，锁定一个范围，并且锁定记录本身  

当查询的索引含有唯一属性时，InnoDB存储引擎会对Next-Key Lock进行优化，将其降级为Record Lock，即仅锁住索引本身，而不是范围。  

### 幻读 aka Phantom Problem  
在默认的事务隔离级别下，即REPEATABLE READ下，InnoDB存储引擎采用Next-Key Locking机制来避免Phantom Problem（幻像问题）。  
Phantom Problem是指在同一事务下，连续执行两次同样的SQL语句可能导致不同的结果，第二次的SQL语句可能会返回之前不存在的行。  

脏读： 读到未提交的数据。  
不可重复读： 读到已提交的数据。InnoDB存储引擎中，通过使用Next-Key Lock算法来避免不可重复读的问题。  
丢失更新，select接下来会更改的数据时用select ... for update。  


### 关于阻塞和死锁  
参数innodb_lock_wait_timeout用来控制等待的时间（默认是50秒），innodb_rollback_on_timeout用来设定是否在等待超时时对进行中的事务进行回滚操作（默认是OFF，代表不回滚）。  

wait-for graph是一种较为主动的死锁检测机制，在每个事务请求锁并发生等待时都会判断是否存在回路，若存在则有死锁，通常来说InnoDB存储引擎选择回滚undo量最小的事务。wait-for graph的死锁检测通常采用深度优先的算法实现。  

## 锁升级  
锁升级（Lock Escalation）是指将当前锁的粒度降低。锁升级会导致并发性能降低。  
InnoDB存储引擎不存在锁升级的问题。因为其不是根据每个记录来产生行锁的，相反，其根据每个事务访问的每个页对锁进行管理的，采用的是位图的方式。因此不管一个事务锁住页中一个记录还是多个记录，其开销通常都是一致的。（暂时没查到位图方式的资料，猜测是优化的数据结构减少了锁的查找与存储开销）。  

## mysql的事务隔离级别  

| 事务隔离级别 \ 是否存在| 丢失修改 | 脏读 | 不可重复读 |  
| :- | :-: |:-: |:-: |  
|读未提交（read-uncommitted）| 否 | 是 | 是 |  
|不可重复读（read-committed）| 否 | 否 | 是 |  
|可重复读（repeatable-read）| 否 | 否 | 否 |  
|串行化（serializable）| 否 | 否 | 否 |  

上表不可重复读的概念是高教第五版本数据库系统概论中的定义。包括1）两次读不一致；2）再次读部分消失；3）再次读多出部分。其中后两者有时也称为幻读。  
Mysql在可重复读（repeatable-read）即默认事务级别下，使用Next-Key Lock锁的算法，因此避免幻读的产生。  


