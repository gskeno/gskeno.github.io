---
layout: post
title: mysql explain索引分析
tags:
- mysql
categories: mysql
description: explain索引分析
---

本文主要介绍mysql explain索引分析。

# explain
## type

- const 主键或者唯一索引，等值查询。
- ALL  表示全表扫描。
- ref   使用了索引，但不是主键也不是唯一索引。但并不是使用了普通索引，就一定是ref，比如还可能是	       range。
- range 索引范围扫描
- index `全索引扫描`，和ALL类似，只不过index是全盘扫描了索引的数据。但当使用`select id from t order by gmt_modified limit 10`，order by 子句时，并非会全索引扫描。

## ref
表示将哪个字段或常量和key列所使用的字段进行比较。
如果ref是一个函数，则使用的值是函数的结果。要想查看是哪个函数，可在EXPLAIN语句之后紧跟一个SHOW WARNING语句。

## filtered

- 查询最终返回的行数 占 引擎扫描的行数 的百分比。
- 值是一个`评估值`，不够精准，只能当作参考。

## key_len

- 使用的索引部分的字节数。如果一个联合索引占用 4+4+4个字节，而key_len为8，表示只使用到了联合索引的前两列。
- (a,b,c)联合索引，当`where a = x and b = y order by c asc limit 100`时，`key_len为8，不含c字段`，这里我理解，c在key_len中没有用处，引擎查询时只检索满足索引的前8个字节的联合索引，c只是用来辅助排序。
## Extra

- Using where
   - 如果我们不是读取表的所有数据，或者不是仅仅通过索引就可以获取所有需要的数据，则会出现using where信息。如`explain SELECT * FROM t1 where id > 5`。
- Using index
   - 仅使用索引树中的信息从表中检索列信息，而不必进行其他查找以读取实际行。当查询仅使用属于单个索引的列时，可以使用此策略。例如：`explain SELECT id FROM t`。
   - 只使用单个索引，不用回表，即可得到查询结果。例如 `explain select  id  from info where status = 1`。
- Using index condition
   - 表示先按条件过滤索引，过滤完索引后找到所有符合索引条件的数据行，随后用 WHERE 子句中的其他条件去过滤这些数据行。比如 `explain select city,name,age from t where city='杭州' order by name limit 1000  ;`只建立了city索引时。
- Using filesort
   - 当利用索引无法进行排序时，出现Using filesort，但并不代表一定要文件磁盘排序。数据较少时从内存排序，否则从磁盘排序。
- Backward index scan
   - 反向扫描
   - mysql 8才支持，在之前，索引扫描默认按照升序。

# 数据准备
创建info表
```sql
DROP   table  if exists  info;

CREATE TABLE info (
    id INT NOT NULL PRIMARY KEY AUTO_INCREMENT,
    status INT NOT NULL,
    type INT NOT NULL,
    gmt_modified BIGINT
);
```

创建存储过程
```sql
CREATE DEFINER=`root`@`localhost` PROCEDURE `random_info_rows`(IN n int)
BEGIN  
  DECLARE i INT DEFAULT 1;
  
  while i < 1000000 do 
    IF i % 10000 = 0 THEN
    	INSERT into info (status,type, gmt_modified) VALUES (1, FLOOR(RAND() * 2 + 1),FLOOR(RAND() * 10000000));
      SET i=i+1;
    ELSEIF i % 10001 = 0 THEN
    	INSERT into info (status,type, gmt_modified) VALUES (2, FLOOR(RAND() * 2 + 1),FLOOR(RAND() * 10000000));
      SET i=i+1;
    ELSE 
      INSERT into info  (status,type, gmt_modified)VALUES(FLOOR(RAND() * 3 + 3), FLOOR(RAND() * 2 + 1), FLOOR(RAND() * 10000000));
      SET i=i+1;
    END IF;
  end while;
END
```
status = 1，2各100条左右；type=1，2各占一半，gmt_modified随机。
# 案例一
表t有一百万条数据，有id, status, type, gmt_modified三列。其中status=1,2分别有百条数据左右，type=1，2各占一半，`问如下查询建立何种索引更加合适`?
> select id from t where type = status = 1 and type = 1 order by gmt_modified desc;


第一直觉，感觉建立(status, type, gmt_modified)的联合索引，原因如下。

- status=1区分度很大，放在联合索引的第一列很合适，
- type作为第二列是为了第三列gmt_modified的排序可以利用联合索引，而不必利用`file sort`(server层，排序数据量小用内存排序，数据量大用磁盘排序) 进行gmt_modified倒排。

以mysql 8 建立(status, type, gmt_modified) 联合索引，explain分析如下:
![](/assets/img/mysql/explain1.png)

- type为ref，表明使用到了普通索引扫描。
- key为idx_status_type_gmtmodified，表明使用了该联合索引
- key_len为8，表示扫描利用到了联合索引的前2列，第3列是用来排序的，对过滤无作用。
- ref位const,const，表示普通索引扫描 status = 1 and type = 1 将status列与常量1进行了比较，type列与常量1进行了比较。
- rows为52，表示扫描了52行。
- filtered为100，表示扫描到的行全部都是返回数据，无一被过滤掉。
- Extra中Using where 表示非读取表全部数据；Using index表示使用了索引; Backward index scan表示用到了索引反向扫描。

再多想一步，由于type区分度不大(1/2)，status=1扫描走索引就能找到满足条件的最终返回行数X的2倍数据行数(2X)。这其中的一半行数，再走聚簇索引扫描过滤以及利用`filesort`倒序排序也是ok的(数据量小，使用内存排序)。

综合上述分析: 两种建立索引的方式似乎各有优劣。

- 建立联合索引，索引列太多，索引存储空间大。而且如果gmt_modified经常变化的化，索引会重建。好处是不二次回表，利用索引排序。
- 建立status索引，需要二次回表走聚簇索引再过滤并利用filesort排序，但业务只需要返回id主键字段，二次扫表耗费时间。好处是节省索引存储空间。

参考
- [https://www.itmuch.com/mysql/explain/#type](https://www.itmuch.com/mysql/explain/#type)
- [https://dev.mysql.com/doc/refman/8.0/en/explain-output.html](https://dev.mysql.com/doc/refman/8.0/en/explain-output.html)
- [https://dev.mysql.com/blog-archive/mysql-8-0-labs-descending-indexes-in-mysql/](https://dev.mysql.com/blog-archive/mysql-8-0-labs-descending-indexes-in-mysql/)
- [https://dev.mysql.com/doc/refman/5.7/en/limit-optimization.html](https://dev.mysql.com/doc/refman/5.7/en/limit-optimization.html)
