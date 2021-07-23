---
title: Mysql 的 upsert 操作
top: false
cover: false
toc: true
mathjax: false
date: 2021-07-23 14:15:48
password:
summary: 无论记录是否存在，只需一句sql解决
categories: 数据库
tags: 
- mysql
- replace
---

当我们在使用 Mysql 时，经常会有 `upsert` 这样的需求：插入一行唯一数据时，若该行数据已存在，则更新该行内容，不存在则插入新的一行。但是 Mysql 没有 MongoDB 中 `upsert` 这样的函数可以直接实现，一般来说会先用 `Count(*)` 查找是否存在该行，然后根据结果执行不通的操作，这样操作就有些繁琐了。  
我想着 Mysql 应该会有对这样的操作进行优化的方法，抱着这一猜想，我查看了官方文档，确实发现有两个函数可以实现这样的功能，下面我们一起来看下:v:。

## 使用 REPLACE

### 语法

替换语句语法：

```sql
REPLACE INTO tab_name VALUES (val, ...)
```

该语句在一般情况下和 `INSERT` 语句的功能类似，插入一行新数据，当遇到主键或唯一索引冲突时，会删除旧行插入新行，实现替换的目的

我们以会员表为例来说明

```sql
CREATE TABLE `member` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `name` varchar(255) DEFAULT NULL,
  `sex` tinyint(1) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

### 更新数据包含主键

```sql
-- 插入一条数据
mysql> REPLACE INTO member VALUES (1, "alice", 1);
Query OK, 1 row affected (0.00 sec)

-- 修改第一条数据的 sex 字段
mysql> mysql> REPLACE INTO member VALUES (1, "alice", 0);
Query OK, 2 rows affected (0.00 sec)

-- 数据库仅有一条数据，替换成功了
mysql> SELECT * FROM member;
+----+-------+------+
| id | name  | sex  |
+----+-------+------+
|  1 | alice |    0 |
+----+-------+------+
1 row in set (0.00 sec)

```

当更新的数据包含主键时，更新同一条数据会发生冲突，从而发生替换。而且，根据返回的影响行数(raws)，我们也可以知道发生了什么操作，当 `raws=1` 时，执行了插入操作，当 `raws>1` 时，执行了替换的操作，为什么是大于1呢，`REPLACE` 的 `raws` 包含删除行数和插入行数，当存在多个唯一索引时，替换会删除多行旧数据，插入一行，因此有时会大于1。

### 更新数据不包含主键

```sql
mysql> REPLACE INTO member (name, sex) VALUES ("brain", 0);
Query OK, 1 row affected (0.00 sec)

mysql> REPLACE INTO member (name, sex) VALUES ("brain", 1);
Query OK, 1 row affected (0.00 sec)

mysql> select * from member;
+----+-------+------+
| id | name  | sex  |
+----+-------+------+
|  1 | alice |    0 |
|  2 | brain |    0 |
|  3 | brain |    1 |
+----+-------+------+
3 rows in set (0.00 sec)
```

当更新数据不包含主键，也没有索引时，仅会发生插入动作。下面我们来看表中存在唯一索引的情况。

### 更新包含唯一索引

将第三条记录删除，并且给 `name` 添加索引

```sql
DELETE FROM `test`.`member` WHERE `id` = 3;
ALTER TABLE `test`.`member` 
ADD UNIQUE INDEX(`name`);
```

现在更新 `ciri` 的信息

```sql
mysql> REPLACE INTO member (name, sex) VALUES ("ciri", 1);
Query OK, 1 row affected (0.00 sec)

mysql> REPLACE INTO member (name, sex) VALUES ("ciri", 0);
Query OK, 2 rows affected (0.00 sec)

mysql> select * from member;
+----+-------+------+
| id | name  | sex  |
+----+-------+------+
|  1 | alice |    0 |
|  2 | brain |    0 |
|  5 | ciri  |    0 |
+----+-------+------+
3 rows in set (0.00 sec)

```

根据影响行数，我们发现第一次更新时插入了新行，第二次更新发生了替换，但是相应的 `id` 也自增了，从 `4` 变成 `5`，因为我们没有指定 `id`，替换操作删除了旧行，插入了新行。因此，:exclamation:我们**使用 `REPLACE` 语句时，必须保证表中有主键或是唯一索引，若 `id` 为自增时，更新数据必须包含 `id`**, 否则查询数据将会返回错误结果。

更多信息请阅读[官方文档](https://dev.mysql.com/doc/refman/5.7/en/replace.html)，下面我们来看第二种方法

## 使用 ON DUPLICATE KEY UPDATE

以上面的会员表结构为例，该语句会先执行 `INSERT` 操作，若 `INSERT` 成功，则后面的 `UPDATE` 语句会被忽略，若插入时发生主键或唯一索引冲突，则执行 `UPDATE` 操作

```sql
INSERT INTO member (id, name, sex) VALUES (1,'alice',0)
  ON DUPLICATE KEY UPDATE sex=1;
-- same as: 
UPDATE member SET sex=1 WHERE id=1;
```

使用 `ON DUPLICATE KEY UPDATE`时，影响行数会有如下几种情况：

- `raws=0`：没有更新数据，
- `raws=1`：将 `VALUES` 数据插入了新行
- `raws=2`：更新了当前行的数据为 `UPDATE` 后面的内容

若要更新多个字段，则用 `,` 拼接多个列赋值语句

该语句和 `REPLACE` 的条件一样，执行操作的表必须有主键或者唯一索引，且 `VALUES` 中要包含对应列

该语句比 `REPLACE` 好的一点是不会删除旧行，仅仅会进行更新和插入操作，因此可以避免误删除造成的数据不完整

```sql
mysql> INSERT INTO member (id, name, sex) VALUES (5, 'ciri' ,1)   
    ->   ON DUPLICATE KEY UPDATE sex=0, name='chris';
Query OK, 2 rows affected (0.00 sec)

mysql> select * from member;
+----+-------+------+
| id | name  | sex  |
+----+-------+------+
|  1 | alice |    0 |
|  2 | brain |    0 |
|  5 | chris |    0 |
+----+-------+------+
3 rows in set (0.00 sec)

```

想了解更多信息请阅读[官方文档](https://dev.mysql.com/doc/refman/5.7/en/insert-on-duplicate.html)

## 总结

当我们需要 `upsert` 操作时，且当前表中有主键或唯一索引，我们就可以使用 `REPLACE` 和 `ON DUPLICATE KEY UPDATE` 语句来进行更新或插入不确定是否存在的记录，但是建议使用 `ON DUPLICATE KEY UPDATE` 语句，可以避免误删除导致的悲剧，特别注意：**自增列必须包含在语句中**。
