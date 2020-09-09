


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

类似问题[leetcode链接](https://leetcode-cn.com/problems/reformat-department-table/)  

```
SELECT s.c_id,s.s_id,s.s_score 
FROM score s
WHERE 2>(
SELECT COUNT( s2.s_score)
FROM score s2 
WHERE s2.`c_id`=s.`c_id` AND s2.`s_score`>s.`s_score`
)
 ORDER BY s.`c_id` ASC, s.`s_score` DESC;
```