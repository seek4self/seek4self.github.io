---
title: Postgres 心得
top: false
cover: false
toc: true
mathjax: false
date: 2022-03-17 15:25:35
password:
summary: 使用 Postgres 的经历
categories: 数据库
tags: 
- Postgres
- SQL
- psql
---

## 使用 Docker 部署

```bash
docker run --name postgres \
    -e POSTGRES_PASSWORD=123456 \
    -e PGDATA=/var/lib/postgresql/data/pgdata \
    -v /opt/data/postgres:/var/lib/postgresql/data \
    -p 5432:5432 -d \
    postgres:14.1-alpine3.15
```

部署后默认创建 `postgres` 数据库，用户名为 `postgres`，密码为 `123456`

若需要启动容器时创建新的数据库，则需要在 `/docker-entrypoint-initdb.d` 文件夹下创建初始化 sql 脚本

```sql
CREATE USER seeker;  -- 创建新用户

CREATE DATABASE db1; -- 创建新数据库
GRANT ALL PRIVILEGES ON DATABASE db1 TO seeker; -- 赋予用户 seeker 访问 db1 的所有权限
```

## psql 命令行访问

容器启动后，可以 `exec` 进入容器，使用自带的 `psql` 客户端命令行访问服务

```bash
psql [option...] [dbname [username]]

$ docker exec -it posgres psql db1 seeker
psql (14.1)
Type "help" for help.

db1=# 
```

psql 选项：

- `-c, --command=COMMAND`：执行一条 SQL 或命令并退出
- `-d, --dbname=DBNAME`：指定要连接的数据库名，(默认: `"root"`)
- `-h, --host=HOSTNAME`：数据库服务地址， (默认: `"local socket"`)
- `-p, --port=PORT`: 数据库服务端口，(默认 `"5432"`)
- `-U, --username=USERNAME`：用户名，(默认 `"root"`)
- `-w, --no-password`: 不提示密码登录，会默认使用 Docekr 环境变量的密码
- `-W, --password`: 强制提示输入密码，

更多命令行应用可以查看 [psql](http://www.postgres.cn/docs/13/app-psql.html)，下面列出常用的几个命令

### 查看所有数据库

```sql
postgres=# \l  -- 列出所有数据库信息
                                 List of databases
   Name    |  Owner   | Encoding |  Collate   |   Ctype    |   Access privileges   
-----------+----------+----------+------------+------------+-----------------------
 postgres  | postgres | UTF8     | en_US.utf8 | en_US.utf8 | 
 template0 | postgres | UTF8     | en_US.utf8 | en_US.utf8 | =c/postgres          +
           |          |          |            |            | postgres=CTc/postgres
...

postgres=# SELECT datname FROM pg_database; -- 使用 sql 查询 数据库名
```

### 切换数据库

```sql
postgres=# \c db1 --  \connect 相当于重新开启一个新的连接，如果登录时设置了 -W 选项，则会提示输入密码
You are now connected to database "db1" as user "postgres".
db1=# 
```

### 查看所有表

```sql
postgres=# \dt -- 列出所有表信息， 后面可以跟 正则表达式
         List of relations
 Schema |     Name     | Type  |  Owner   
--------+--------------+-------+----------
 public | tablename    | table | postgres
 ...

postgres=# select tablename from pg_tables where schemaname = 'public'; -- 使用 sql 查询 数据表名
```

### 查看表结构

```sql
postgres=# \d channels  -- 查看表结构和索引信息
                            Table "public.channels"
 Column |  Type  | Collation | Nullable |               Default                
--------+--------+-----------+----------+--------------------------------------
 id     | bigint |           | not null | nextval('channels_id_seq'::regclass)
 name   | text   |           |          | 
Indexes:
    "channels_pkey" PRIMARY KEY, btree (id)
```

## 遇到的一些问题

### 迁移数据表，自增id失效

当从一个数据库中拷贝一张表到另一个数据库，数据虽然过去了，但是自增 id 是从 1 开始的，这时候需要将自增 id 改为当前表的最后一个 id, Postgres 也提供了 `setval` 函数可以修改

```sql
SELECT setval('table_id_seq'::regclass, (select max(id) from table));
```

### 表不存在

服务启动前没有预先建表，当查询时，数据表不存在，Postgres 会报错 `ERROR: relation "table" does not exist (SQLSTATE 42P01)`，为了避免因为这个错误影响结果，可以预先在系统表里查找该表是否存在

```sql
select count(relname) from pg_class where relname = 'tablename';
```
