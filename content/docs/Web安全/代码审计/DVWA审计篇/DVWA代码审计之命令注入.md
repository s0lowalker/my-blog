---
title: DVWA代码审计之命令注入
date: 2026-03-29
tags: [代码审计]
categories: [Web安全] 
excerpt: DVWA之命令注入代码审计
---

# DVWA代码审计之命令注入

## 原理

在操作系统中，&，&&，|，||都可以作为命令连接符使用，用户通过浏览器提交执行命令，由于服务器端没有对执行函数进行过滤，从而造成可以执行危险命令。

PHP的主要命令执行函数有：`system`、`exec`、`passthru`、`shell_exec`。反引号同样能够执行命令，但不是函数。

### 连接符的使用

`cmd1 & cmd2`：不管`cmd1`执行成功与否，都执行`cmd2`，即`cmd1`和`cmd2`都执行。

`cmd1 && cmd2`：先执行`cmd1`，成功后再执行`cmd2`，若`cmd1`执行失败，`cmd2`也不执行。

`cmd1 | cmd2`：把`cmd1`的输出作为`cmd2`的输入。

`cmd1 || cmd2`：若`cmd1`执行失败，再执行`cmd2`；反之，`cmd2`不执行。

## 等级low

### 源码

```php
<?php

if( isset( $_POST[ 'Submit' ]  ) ) {
	// Get input
	$target = $_REQUEST[ 'ip' ];

	// Determine OS and execute the ping command.
	if( stristr( php_uname( 's' ), 'Windows NT' ) ) {
		// Windows
		$cmd = shell_exec( 'ping  ' . $target );
	}
	else {
		// *nix
		$cmd = shell_exec( 'ping  -c 4 ' . $target );
	}

	// Feedback for the end user
	$html .= "<pre>{$cmd}</pre>";
}

?>
```

#### 函数简介

`php_uname()`：返回运行PHP的操作系统的描述，当参数为s时表示指定只返回操作系统名称。

`stristr(string,search,before_search)`：这是PHP中的字符串查找函数，用于查找字符串中第一次出现的位置，不分大小写。

`shell_exec()`：这是PHP中用于执行系统命令的函数，并返回命令的完整输出。

### 漏洞点

在源码中，`$target`直接拼接到`shell_exec`之中，这是我们用上面讲的命令连接符即可执行系统命令，获取敏感信息。

## 等级medium

### 源码

```php
<?php

if( isset( $_POST[ 'Submit' ]  ) ) {
	// Get input
	$target = $_REQUEST[ 'ip' ];

	// Set blacklist
	$substitutions = array(
		'&&' => '',
		';'  => '',
	);

	// Remove any of the charactars in the array (blacklist).
	$target = str_replace( array_keys( $substitutions ), $substitutions, $target );

	// Determine OS and execute the ping command.
	if( stristr( php_uname( 's' ), 'Windows NT' ) ) {
		// Windows
		$cmd = shell_exec( 'ping  ' . $target );
	}
	else {
		// *nix
		$cmd = shell_exec( 'ping  -c 4 ' . $target );
	}

	// Feedback for the end user
	$html .= "<pre>{$cmd}</pre>";
}

?>
```

### 与low的区别

与low相比，medium做了一点简单的防护，这里使用了黑名单：

```php
$substitutions = array(
		'&&' => '',
		';'  => '',
	);
```

ban掉了&&和“;”，但是上面所讲的命令连接符依然有可以利用的。

## 等级high

### 源码

```php
<?php

if( isset( $_POST[ 'Submit' ]  ) ) {
	// Get input
	$target = trim($_REQUEST[ 'ip' ]);

	// Set blacklist
	$substitutions = array(
		'&'  => '',
		';'  => '',
		'| ' => '',
		'-'  => '',
		'$'  => '',
		'('  => '',
		')'  => '',
		'`'  => '',
		'||' => '',
	);

	// Remove any of the charactars in the array (blacklist).
	$target = str_replace( array_keys( $substitutions ), $substitutions, $target );

	// Determine OS and execute the ping command.
	if( stristr( php_uname( 's' ), 'Windows NT' ) ) {
		// Windows
		$cmd = shell_exec( 'ping  ' . $target );
	}
	else {
		// *nix
		$cmd = shell_exec( 'ping  -c 4 ' . $target );
	}

	// Feedback for the end user
	$html .= "<pre>{$cmd}</pre>";
}

?>

```

### 防御手法

这里的防御措施依旧是使用黑名单，这里的黑名单看起来是很全面的，但是这里的`'| '`是在|后面加上空格的，我们可以用`' |'`来绕过。当然，我在注入的时候习惯性的会两边加上空格。

## 等级impossible

### 源码

```php
<?php

if( isset( $_POST[ 'Submit' ]  ) ) {
	// Check Anti-CSRF token
	checkToken( $_REQUEST[ 'user_token' ], $_SESSION[ 'session_token' ], 'index.php' );

	// Get input
	$target = $_REQUEST[ 'ip' ];
	$target = stripslashes( $target );

	// Split the IP into 4 octects
	$octet = explode( ".", $target );

	// Check IF each octet is an integer
	if( ( is_numeric( $octet[0] ) ) && ( is_numeric( $octet[1] ) ) && ( is_numeric( $octet[2] ) ) && ( is_numeric( $octet[3] ) ) && ( sizeof( $octet ) == 4 ) ) {
		// If all 4 octets are int's put the IP back together.
		$target = $octet[0] . '.' . $octet[1] . '.' . $octet[2] . '.' . $octet[3];

		// Determine OS and execute the ping command.
		if( stristr( php_uname( 's' ), 'Windows NT' ) ) {
			// Windows
			$cmd = shell_exec( 'ping  ' . $target );
		}
		else {
			// *nix
			$cmd = shell_exec( 'ping  -c 4 ' . $target );
		}

		// Feedback for the end user
		$html .= "<pre>{$cmd}</pre>";
	}
	else {
		// Ops. Let the user name theres a mistake
		$html .= '<pre>ERROR: You have entered an invalid IP.</pre>';
	}
}

// Generate Anti-CSRF token
generateSessionToken();

?>
```

### 关键函数解析

#### `explode()`

这是PHP中用于将字符串分割成数组的函数。

##### 基本语法

```php
explode(string $separator, string $string, int $limit = PHP_INT_MAX): array
```

`$separator`：分隔符，用于指定在哪里分割字符串

`$string`：要分割的原始字符串

`$limit`(可选)：限制返回数组的最大元素个数

### 防御逻辑

使用`explode()`来把我们传入的参数进行分割，然后用`is_numeric`判断是否为纯数字，并且判断数组大小是不是4，然后进行ip的重构。当我们使用命令连接符时，在纯数字的验证时会出错，从而阻止执行其他命令的可能。



