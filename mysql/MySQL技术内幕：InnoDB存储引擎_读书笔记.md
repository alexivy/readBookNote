基于原书第二版。  

- [第5章 索引与算法（待完善）](#第5章-索引与算法待完善)
- [index](#index)
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

# 第5章 索引与算法（待完善）  
# index  
看目录感觉和《MySQL技术内幕：SQL编程》中的 第九章 索引内容差不多，相近部分结合[此处](/mysql/MySQL技术内幕：SQL编程_读书笔记.md)查看。此处只补充一些不同的部分。  

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


