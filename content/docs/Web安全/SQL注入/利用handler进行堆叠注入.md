---
title: 堆叠注入之利用handler语句获取数据
date: 2026-03-24
tags: [数据库]
categories: [Web安全] 
excerpt: 在堆叠注入中利用handler语句获取数据
---

# BUUCTF之[GYCTF2020]Blacklist

## handler语句

HANDLER 是 MySQL 中用于直接访问表存储引擎接口的语句，类似于文件操作：

```mysql
-- 打开表
HANDLER table_name OPEN [AS alias];

-- 读取数据
HANDLER table_name READ FIRST;     -- 读取第一行
HANDLER table_name READ NEXT;      -- 读取下一行
HANDLER table_name READ {索引名} {操作};

-- 关闭表
HANDLER table_name CLOSE;
```

### 与union的对比

#### handler的优势

- 不需要知道列数，即不需要`order by`来查询
- 不受`select`限制，如果`select`被ban了也能查询到数据
- 访问数据更灵活，可以精确控制读取到哪一行

## 注入过程

![1](/images/screenshots/handler-01.png)

首先测试一下是什么类型的注入，输入`1'`报错，即为字符型注入。

![2](/images/screenshots/handler-02.png)

然后测试了`1' order by 3`，没有这一列，测试`order by 2`则通过。

尝试经典的联合查询注入，结果回显：

![3](/images/screenshots/handler-03.png)

回显了waf，`select`等查询字符被过滤。大概率是堆叠注入。然后测试`show`语句，输入`1';show databases;#`，回显：

![4](/images/screenshots/handler-04.png)

确认是堆叠注入，也能看到这些数据库名。然后看一下当前数据库的数据表，输入`1';show tables;#`，回显：

![5](/images/screenshots/handler-05.png)

此时很明显了，flag就在这。然后查一下FlagHere这个表的字段，输入`1';show columns from FlagHere;#`，回显：

![6](/images/screenshots/handler-06.png)

此时我们需要查询到flag字段里面的数据，这时就可以用上面讲到的`handler`语句来获取数据，输入`1';handler FlagHere open;handler FlagHere read first;handler FlagHere close;#`，即可回显flag。

## 总结

一般来说如果是有回显的SQL注入，而`select`之类的查询语句又被ban了，很大可能是堆叠注入。这可能需要对数据库有更深的理解，有机会的话还是可以再多学习一点数据库的知识。
