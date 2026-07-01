---
title: DVWA代码审计之SQL注入
date: 2026-03-14
tags: [代码审计]
categories: [Web安全] 
excerpt: DVWA之SQL注入代码审计
---

# DVWA代码审计之SQL注入篇

## 等级low

### 源码功能

```php
<?php

if( isset( $_REQUEST[ 'Submit' ] ) ) {
	// 获取输入
	$id = $_REQUEST[ 'id' ];

	// 数据库查询
	$query  = "SELECT first_name, last_name FROM users WHERE user_id = '$id';";
	$result = mysqli_query($GLOBALS["___mysqli_ston"],  $query ) or die( '<pre>' . ((is_object($GLOBALS["___mysqli_ston"])) ? mysqli_error($GLOBALS["___mysqli_ston"]) : (($___mysqli_res = mysqli_connect_error()) ? $___mysqli_res : false)) . '</pre>' );

	// 处理查询结果
	while( $row = mysqli_fetch_assoc( $result ) ) {
		// 获取对应的值
		$first = $row["first_name"];
		$last  = $row["last_name"];

		// 返回给用户
		$html .= "<pre>ID: {$id}<br />First name: {$first}<br />Surname: {$last}</pre>";
	}

	mysqli_close($GLOBALS["___mysqli_ston"]);
}

?>
```

### 安全隐患

1. 源码使用`$_REQUEST`这个超全局数组接收用户输入，但没有进行任何过滤，这就给了用户操作的空间，通过构造特定的SQL查询语句从而窃取到数据库内的信息。
2. `$query  = "SELECT first_name, last_name FROM users WHERE user_id = '$id';";`这段代码是我们利用的核心，在这个语句中，`$id`这个变量直接插入到查询语句中，导致了**用户输入被当成SQL语句**执行。

### 漏洞成因

SQL注入这类漏洞的核心问题，就是没有把数据和代码分离，使得用户输入被当成SQL代码的一部分执行。

## 等级medium

### 源码功能

```php
<?php

if( isset( $_POST[ 'Submit' ] ) ) {
	// Get input
	$id = $_POST[ 'id' ];
	// 这里的使用mysqli_real_escape_string来进行特殊字符的转义
	$id = mysqli_real_escape_string($GLOBALS["___mysqli_ston"], $id);

	$query  = "SELECT first_name, last_name FROM users WHERE user_id = $id;";
	$result = mysqli_query($GLOBALS["___mysqli_ston"], $query) or die( '<pre>' . mysqli_error($GLOBALS["___mysqli_ston"]) . '</pre>' );

	// Get results
	while( $row = mysqli_fetch_assoc( $result ) ) {
		// Display values
		$first = $row["first_name"];
		$last  = $row["last_name"];

		// Feedback for end user
		$html .= "<pre>ID: {$id}<br />First name: {$first}<br />Surname: {$last}</pre>";
	}

}

// This is used later on in the index.php page
// Setting it here so we can close the database connection in here like in the rest of the source scripts
$query  = "SELECT COUNT(*) FROM users;";
$result = mysqli_query($GLOBALS["___mysqli_ston"],  $query ) or die( '<pre>' . ((is_object($GLOBALS["___mysqli_ston"])) ? mysqli_error($GLOBALS["___mysqli_ston"]) : (($___mysqli_res = mysqli_connect_error()) ? $___mysqli_res : false)) . '</pre>' );
$number_of_rows = mysqli_fetch_row( $result )[0];

mysqli_close($GLOBALS["___mysqli_ston"]);
?>

```

### 安全隐患

尽管使用了`mysqli_real_escape_string`进行字符串的转义，但是查询语句进行了改变，变成`$query  = "SELECT first_name, last_name FROM users WHERE user_id = $id;";`在low等级中，查询语句是有单引号包裹的，在注入时需要用单引号闭合。但在这个情况进行转义其实并没有作用，因为查询语句中没有引号需要，直接进行数字型注入即可。

### 宽字节注入

观察源码后，我有个疑问，如果在查询语句`$query  = "SELECT first_name, last_name FROM users WHERE user_id = $id;";`中的`$id`处加上单引号怎么实现SQL注入。

这就涉及到一种SQL注入手法：宽字节注入。

#### 宽字节注入原理

当数据库使用GBK、BIG5等多字节字符集时，反斜杠会和前面的字符组合成一个合法汉字，导致转义失败。

当我们输入`%df' union`时，经过转义后为`%df\`，这在GBK编码下是一个合法汉字，导致转义失败，从而实现注入。

## 等级high

### 代码功能

```php
<?php

if( isset( $_SESSION [ 'id' ] ) ) {
	// Get input
	$id = $_SESSION[ 'id' ];

	// Check database
	$query  = "SELECT first_name, last_name FROM users WHERE user_id = '$id' LIMIT 1;";
	$result = mysqli_query($GLOBALS["___mysqli_ston"], $query ) or die( '<pre>Something went wrong.</pre>' );

	// Get results
	while( $row = mysqli_fetch_assoc( $result ) ) {
		// Get values
		$first = $row["first_name"];
		$last  = $row["last_name"];

		// Feedback for end user
		$html .= "<pre>ID: {$id}<br />First name: {$first}<br />Surname: {$last}</pre>";
	}

	((is_null($___mysqli_res = mysqli_close($GLOBALS["___mysqli_ston"]))) ? false : $___mysqli_res);		
}

?>
```

### 安全隐患

相比于前面两个，这个源码的安全度高了很多，因为获取的参数是从session中获取的，而session是从服务器端获取的，攻击者无法直接控制，因此更加安全。

在DVWA这个SQL注入类漏洞里面有个文件叫做session_input.php，源码为：

```php
<?php

define( 'DVWA_WEB_PAGE_TO_ROOT', '../../' );
require_once DVWA_WEB_PAGE_TO_ROOT . 'dvwa/includes/dvwaPage.inc.php';

dvwaPageStartup( array( 'authenticated', 'phpids' ) );

$page = dvwaPageNewGrab();
$page[ 'title' ] = 'SQL Injection Session Input' . $page[ 'title_separator' ].$page[ 'title' ];

if( isset( $_POST[ 'id' ] ) ) {
	$_SESSION[ 'id' ] =  $_POST[ 'id' ];
	//$page[ 'body' ] .= "Session ID set!<br /><br /><br />";
	$page[ 'body' ] .= "Session ID: {$_SESSION[ 'id' ]}<br /><br /><br />";
	$page[ 'body' ] .= "<script>window.opener.location.reload(true);</script>";
}

$page[ 'body' ] .= "
<form action=\"#\" method=\"POST\">
	<input type=\"text\" size=\"15\" name=\"id\">
	<input type=\"submit\" name=\"Submit\" value=\"Submit\">
</form>
<hr />
<br />

<button onclick=\"self.close();\">Close</button>";

dvwaSourceHtmlEcho( $page );

?>
```

观察源码发现，有`$_SESSION[ 'id' ] =  $_POST[ 'id' ];`，这样危害就大了，我们是可以直接控制session的内容的，这就给我们注入的机会。

## 等级impossible

### 源码

```php
<?php

if( isset( $_GET[ 'Submit' ] ) ) {
	// Check Anti-CSRF token
	checkToken( $_REQUEST[ 'user_token' ], $_SESSION[ 'session_token' ], 'index.php' );

	// Get input
	$id = $_GET[ 'id' ];

	// Was a number entered?
	if(is_numeric( $id )) {
		// Check the database
		$data = $db->prepare( 'SELECT first_name, last_name FROM users WHERE user_id = (:id) LIMIT 1;' );
		$data->bindParam( ':id', $id, PDO::PARAM_INT );
		$data->execute();
		$row = $data->fetch();

		// Make sure only 1 result is returned
		if( $data->rowCount() == 1 ) {
			// Get values
			$first = $row[ 'first_name' ];
			$last  = $row[ 'last_name' ];

			// Feedback for end user
			$html .= "<pre>ID: {$id}<br />First name: {$first}<br />Surname: {$last}</pre>";
		}
	}
}

// Generate Anti-CSRF token
generateSessionToken();

?>
```

### 参数化查询

目前对于SQL注入最常用的防护方法就是参数化查询，也叫预编译。

普通的SQL语句执行时都会经过七个步骤，即**接收SQL语句、语法解析、语义检查、权限检查、生成执行计划、执行查询、返回结果**，而下一次执行SQL语句时会重复这样一个过程，效率低。

而预编译执行则是在生成执行计划后保存执行计划，当我们第二次执行时直接使用保存的计划。

#### 为什么参数化查询能够防御SQL注入

我们在编写源码时已经确定了我们的查询语句，这其实是一个模板，经过预编译后编译好的“骨架”已经被保存下来，正在等待数据。此时对于我们输入的任何数据，在数据库看来都只是数据，因为我们的查询语句的框架已经确定，所以即使我们尝试一般的SQL注入语句也不会被解析成SQL语句并执行。

### PHP+MySQL的防御代码

以这里的DVWA为例：

```php
$data = $db->prepare( 'SELECT first_name, last_name FROM users WHERE user_id = (:id) LIMIT 1;' );
$data->bindParam( ':id', $id, PDO::PARAM_INT );
$data->execute();
$row = $data->fetch();  //获取查询结果
```

这是一个典型的参数化查询的代码样例。

首先用`is_numeric`这个PHP内置函数检查是否`$id`这个变量是否为数字，已经是一层防护。

`$data = $db->prepare( 'SELECT first_name, last_name FROM users WHERE user_id = (:id) LIMIT 1;' );`这段代码则是实现预编译的关键，利用PDO的`prepare`方法把其中的SQL语句进行预编译，`(:id)`则是一个占位符。这个方法执行后会返回一个对象。

`$data->bindParam( ':id', $id, PDO::PARAM_INT );`这段代码则是绑定参数并指定参数类型。这是参数化查询的核心步骤。

`$data->execute();`这段代码则让我们的查询语句执行起来。

## 总结

这是我开始学习代码审计的第一站，在阅读DVWA的SQL注入的核心源码之后我还是收获良多的，以前对于SQL注入只是知道怎么打，对于参数化查询也只是有这么一个概念，并不知道这么实现，现在通过代码审计我也知道怎么防御SQL注入了。

对于一个安全人员来说，只知道怎么打不知道怎么防是不合格的，之后我还要继续努力，补全我的知识盲区。
