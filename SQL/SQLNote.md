
- [sql语句](#sql语句)
  - [竖表变横表](#竖表变横表)
  - [查询各科目成绩高的前几名](#查询各科目成绩高的前几名)
- [分库分表](#分库分表)

# sql语句

## 竖表变横表  
score表（s_id,c_id,s_score）三列，分别表示（学号，课程号，成绩）。  

要求结果是学生的所有成绩在一行中。  

类似问题[leetcode链接](https://leetcode-cn.com/problems/reformat-department-table/)  

方法1：表连接查询  
```
这种方法不正确，无法列出某科目无成绩的学生。
SELECT s1.s_id ,s1.`s_score` AS "01",s2.`s_score` AS "02",s3.`s_score` AS "03" 
FROM score AS s1  JOIN score AS s2 ON s1.`s_id`=s2.`s_id` AND s1.`c_id`=01 AND s2.`c_id`=02 
JOIN score AS s3 ON s1.`s_id`=s3.`s_id` AND s3.`c_id`=03;
不能使用外连接。
需连接多张表。
```

方法2：聚集函数查询  
```
SELECT s_id ,
MAX(IF(`c_id`='01',s_score,NULL)) "01",
MAX(IF(`c_id`='02',s_score,NULL)) "02",
MAX(IF(`c_id`='03',s_score,NULL)) "03"
FROM score 
GROUP BY s_id;

类似的有：
SELECT s_id ,
SUM(CASE `c_id` WHEN '01' THEN s_score END ) '01',
SUM(CASE `c_id` WHEN '02' THEN s_score END ) '01',
SUM(CASE `c_id` WHEN '03' THEN s_score END ) '01'
FROM score 
GROUP BY s_id;

```

## 查询各科目成绩高的前几名  


score表（s_id,c_id,s_score）三列，分别表示（学号，课程号，成绩）。  

查询各门成绩前2的学生（成绩相同的也显示）  

类似问题[leetcode链接](https://leetcode-cn.com/problems/reformat-department-table/)  

```
SELECT s.c_id,s.s_id,s.s_score 
FROM score s
WHERE 2>(
SELECT COUNT(distinct s2.s_score)
FROM score s2 
WHERE s2.`c_id`=s.`c_id` AND s2.`s_score`>s.`s_score`
)
 ORDER BY s.`c_id` ASC, s.`s_score` DESC;
```

```
查询中国各地区人口前三的城市。表是MySQL自带的World库中的city
SELECT c1.* FROM city AS c1 WHERE c1.`CountryCode` = 'CHN' AND  3> (
SELECT COUNT(c2.`Population`) FROM city AS c2 
WHERE c2.`CountryCode`=c1.`CountryCode` AND c1.`District`=c2.`District` AND c2.`Population`>c1.`Population`
) ORDER BY c1.`District`,c1.`Population`;
```

```
SELECT c1.* FROM city AS c1 WHERE c1.district IN 
(SELECT district FROM city WHERE countrycode='CHN' GROUP BY district HAVING SUM(Population)>5000000
)
 AND  3> (
SELECT COUNT(c2.`Population`) FROM city AS c2 
WHERE c2.`CountryCode`=c1.`CountryCode` AND c1.`District`=c2.`District` AND c2.`Population`>c1.`Population`
)   
ORDER BY c1.`District`,c1.`Population` DESC;
```


# 分库分表  

[参考连接](https://blog.csdn.net/weixin_44062339/article/details/100491744)  

整体思路降低单一库单一表的数据量（存储量以及访问量）。  

通常包括：垂直分表、垂直分库、水平分库、水平分表四种方式。  

***垂直分表***  
将一个表按照字段分成多表，每个表存储其中一部分字段。  

带来的提升是：  
避免IO争抢并减少锁表的几率，执行部分查询时各表间互不影响。  
充分发挥热门数据的操作效率，不分割时操作较费时的字段会影响整张表的性能，分割后影响的范围变小。  

拆分原则：  
把不常用的字段单独放在一张表；  
把text，blob等大字段拆分出来放在附表中；  
经常组合查询的列放在一张表中；  

***垂直分库***  
垂直分库是指按照业务将表进行分类，分布到不同的数据库上面，每个库可以放在不同的服务器上，它的核心理念是专库专用。  

相比于垂直分表的优点有：降低了对机器的压力。库内垂直分表只解决了单一表数据量过大的问题，但没有将表分布到不同的服务器上，因此每个表还是竞争同一个物理机的CPU、内存、网络IO、磁盘。  

它带来的提升是：
解决业务层面的耦合，业务清晰；  
能对不同业务的数据进行分级管理、维护、监控、扩展等；  
高并发场景下，垂直分库一定程度的提升IO、数据库连接数、降低单机硬件资源的瓶颈；  

***水平分库***  
水平分库是把同一个表的数据按一定规则拆到不同的数据库中，每个库可以放在不同的服务器上。  

它带来的提升是：  
解决了单库大数据，高并发的性能瓶颈；  
提高了系统的稳定性及可用性；  

***水平分表***  
水平分表是在同一个数据库内，把同一个表的数据按一定规则拆到多个表中。  
它带来的提升是：  
优化单一表数据量过大而产生的性能问题；  
避免IO争抢并减少锁表的几率；  
解决了单一表数据量过大的问题，分出来的小表中只包含一部分数据，从而使得单个表的数据量变小，提高检索性能。  


总结  
垂直分表：可以把一个宽表的字段按访问频次、是否是大字段的原则拆分为多个表，这样既能使业务清晰，还能提升部分性能。拆分后，尽量从业务角度避免联查，否则性能方面将得不偿失。  

垂直分库：可以把多个表按业务耦合松紧归类，分别存放在不同的库，这些库可以分布在不同服务器，从而使访问压力被多服务器负载，大大提升性能，同时能提高整体架构的业务清晰度，不同的业务库可根据自身情况定制优化方案。但是它需要解决跨库带来的所有复杂问题。  

水平分库：可以把一个表的数据(按数据行)分到多个不同的库，每个库只有这个表的部分数据，这些库可以分布在不同服务器，从而使访问压力被多服务器负载，大大提升性能。它不仅需要解决跨库带来的所有复杂问题，还要解决数据路由的问题(数据路由问题后边介绍)。  

水平分表：可以把一个表的数据(按数据行)分到多个同一个数据库的多张表中，每个表只有这个表的部分数据，这样做能小幅提升性能，它仅仅作为水平分库的一个补充优化。  

一般来说，在系统设计阶段就应该根据业务耦合松紧来确定垂直分库，垂直分表方案，在数据量及访问压力不是特别大的情况，首先考虑缓存、读写分离、索引技术等方案。若数据量极大，且持续增长，再考虑水平分库水平分表方案。  