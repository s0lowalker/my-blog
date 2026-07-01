---
title: DVWA代码审计之不安全的验证码
date: 2026-03-21
tags: [代码审计]
categories: [Web安全] 
excerpt: DVWA之不安全的验证码漏洞代码审计
---

# DVWA代码审计之不安全的验证码

## 原理

简单来说，这个漏洞的成因是：开发者把验证码的“验证”和密码的“修改”两个步骤在服务器端完全分开，从而导致了业务流程上的逻辑漏洞。

一般正常的逻辑是：如果验证码正确，我们才能修改密码，否则不能修改。如果把这两个步骤分开，就变成：当你提交验证码服务器验证通过后，让你重新发一个请求来修改密码，然后用户提交了新密码，而服务器不再验证验证码。这样就造成了逻辑上的“脱节”，如果我们精心地构造一个“修改密码”的请求，假装你已经通过验证，就能绕过直接修改。

## 等级low

### 源码

```php
<?php

if( isset( $_POST[ 'Change' ] ) && ( $_POST[ 'step' ] == '1' ) ) {
	// Hide the CAPTCHA form
	$hide_form = true;

	// Get input
	$pass_new  = $_POST[ 'password_new' ];
	$pass_conf = $_POST[ 'password_conf' ];

	// Check CAPTCHA from 3rd party
	$resp = recaptcha_check_answer(
		$_DVWA[ 'recaptcha_private_key'],
		$_POST['g-recaptcha-response']
	);

	// Did the CAPTCHA fail?
	if( !$resp ) {
		// What happens when the CAPTCHA was entered incorrectly
		$html     .= "<pre><br />The CAPTCHA was incorrect. Please try again.</pre>";
		$hide_form = false;
		return;
	}
	else {
		// CAPTCHA was correct. Do both new passwords match?
		if( $pass_new == $pass_conf ) {
			// Show next stage for the user
			$html .= "
				<pre><br />You passed the CAPTCHA! Click the button to confirm your changes.<br /></pre>
				<form action=\"#\" method=\"POST\">
					<input type=\"hidden\" name=\"step\" value=\"2\" />
					<input type=\"hidden\" name=\"password_new\" value=\"{$pass_new}\" />
					<input type=\"hidden\" name=\"password_conf\" value=\"{$pass_conf}\" />
					<input type=\"submit\" name=\"Change\" value=\"Change\" />
				</form>";
		}
		else {
			// Both new passwords do not match.
			$html     .= "<pre>Both passwords must match.</pre>";
			$hide_form = false;
		}
	}
}

if( isset( $_POST[ 'Change' ] ) && ( $_POST[ 'step' ] == '2' ) ) {
	// Hide the CAPTCHA form
	$hide_form = true;

	// Get input
	$pass_new  = $_POST[ 'password_new' ];
	$pass_conf = $_POST[ 'password_conf' ];

	// Check to see if both password match
	if( $pass_new == $pass_conf ) {
		// They do!
		$pass_new = ((isset($GLOBALS["___mysqli_ston"]) && is_object($GLOBALS["___mysqli_ston"])) ? mysqli_real_escape_string($GLOBALS["___mysqli_ston"],  $pass_new ) : ((trigger_error("[MySQLConverterToo] Fix the mysql_escape_string() call! This code does not work.", E_USER_ERROR)) ? "" : ""));
		$pass_new = md5( $pass_new );

		// Update database
		$insert = "UPDATE `users` SET password = '$pass_new' WHERE user = '" . dvwaCurrentUser() . "';";
		$result = mysqli_query($GLOBALS["___mysqli_ston"],  $insert ) or die( '<pre>' . ((is_object($GLOBALS["___mysqli_ston"])) ? mysqli_error($GLOBALS["___mysqli_ston"]) : (($___mysqli_res = mysqli_connect_error()) ? $___mysqli_res : false)) . '</pre>' );

		// Feedback for the end user
		$html .= "<pre>Password Changed.</pre>";
	}
	else {
		// Issue with the passwords matching
		$html .= "<pre>Passwords did not match.</pre>";
		$hide_form = false;
	}

	((is_null($___mysqli_res = mysqli_close($GLOBALS["___mysqli_ston"]))) ? false : $___mysqli_res);
}

?>
```

### 漏洞成因

这里有两个step，就是把两个步骤分开了，造成了漏洞。

首先，经过`$resp = recaptcha_check_answer($_DVWA[ 'recaptcha_private_key'],$_POST['g-recaptcha-response']);`这一步检验验证码后，如果验证码正确，服务器会返回一个新的表单且包含一个隐藏字段`step=2`并保存了我们的密码，并没有立即修改密码。

然后我们点击确认后，当step=2时进行修改密码的操作，但是此时没有进行验证，只要后端收到`step=2`的请求就会修改密码。

### 利用手法

抓包之后我们把请求体里的`step=1`修改为`step=2`即可。

## 等级medium

### 源码

```php
<?php

if( isset( $_POST[ 'Change' ] ) && ( $_POST[ 'step' ] == '1' ) ) {
	// Hide the CAPTCHA form
	$hide_form = true;

	// Get input
	$pass_new  = $_POST[ 'password_new' ];
	$pass_conf = $_POST[ 'password_conf' ];

	// Check CAPTCHA from 3rd party
	$resp = recaptcha_check_answer(
		$_DVWA[ 'recaptcha_private_key' ],
		$_POST['g-recaptcha-response']
	);

	// Did the CAPTCHA fail?
	if( !$resp ) {
		// What happens when the CAPTCHA was entered incorrectly
		$html     .= "<pre><br />The CAPTCHA was incorrect. Please try again.</pre>";
		$hide_form = false;
		return;
	}
	else {
		// CAPTCHA was correct. Do both new passwords match?
		if( $pass_new == $pass_conf ) {
			// Show next stage for the user
			$html .= "
				<pre><br />You passed the CAPTCHA! Click the button to confirm your changes.<br /></pre>
				<form action=\"#\" method=\"POST\">
					<input type=\"hidden\" name=\"step\" value=\"2\" />
					<input type=\"hidden\" name=\"password_new\" value=\"{$pass_new}\" />
					<input type=\"hidden\" name=\"password_conf\" value=\"{$pass_conf}\" />
					<input type=\"hidden\" name=\"passed_captcha\" value=\"true\" />
					<input type=\"submit\" name=\"Change\" value=\"Change\" />
				</form>";
		}
		else {
			// Both new passwords do not match.
			$html     .= "<pre>Both passwords must match.</pre>";
			$hide_form = false;
		}
	}
}

if( isset( $_POST[ 'Change' ] ) && ( $_POST[ 'step' ] == '2' ) ) {
	// Hide the CAPTCHA form
	$hide_form = true;

	// Get input
	$pass_new  = $_POST[ 'password_new' ];
	$pass_conf = $_POST[ 'password_conf' ];

	// Check to see if they did stage 1
	if( !$_POST[ 'passed_captcha' ] ) {
		$html     .= "<pre><br />You have not passed the CAPTCHA.</pre>";
		$hide_form = false;
		return;
	}

	// Check to see if both password match
	if( $pass_new == $pass_conf ) {
		// They do!
		$pass_new = ((isset($GLOBALS["___mysqli_ston"]) && is_object($GLOBALS["___mysqli_ston"])) ? mysqli_real_escape_string($GLOBALS["___mysqli_ston"],  $pass_new ) : ((trigger_error("[MySQLConverterToo] Fix the mysql_escape_string() call! This code does not work.", E_USER_ERROR)) ? "" : ""));
		$pass_new = md5( $pass_new );

		// Update database
		$insert = "UPDATE `users` SET password = '$pass_new' WHERE user = '" . dvwaCurrentUser() . "';";
		$result = mysqli_query($GLOBALS["___mysqli_ston"],  $insert ) or die( '<pre>' . ((is_object($GLOBALS["___mysqli_ston"])) ? mysqli_error($GLOBALS["___mysqli_ston"]) : (($___mysqli_res = mysqli_connect_error()) ? $___mysqli_res : false)) . '</pre>' );

		// Feedback for the end user
		$html .= "<pre>Password Changed.</pre>";
	}
	else {
		// Issue with the passwords matching
		$html .= "<pre>Passwords did not match.</pre>";
		$hide_form = false;
	}

	((is_null($___mysqli_res = mysqli_close($GLOBALS["___mysqli_ston"]))) ? false : $___mysqli_res);
}

?>
```

### 漏洞成因

简单看一下源码，发现与low相比，medium做出的防护是在修改密码时增加了一个`if(!$_POST['passed_captcha'])`来进行是否通过验证，同时在上面的表单中也增加了对应的字段。

### 利用手法

利用的时候和low的方法没有区别，只需要在请求体添加`passed_captcha=true`即可，因为后端是这样校验的。

## 等级high

### 源码

```php
<?php

if( isset( $_POST[ 'Change' ] ) ) {
	// Hide the CAPTCHA form
	$hide_form = true;

	// Get input
	$pass_new  = $_POST[ 'password_new' ];
	$pass_conf = $_POST[ 'password_conf' ];

	// Check CAPTCHA from 3rd party
	$resp = recaptcha_check_answer(
		$_DVWA[ 'recaptcha_private_key' ],
		$_POST['g-recaptcha-response']
	);

	if (
		$resp || 
		(
			$_POST[ 'g-recaptcha-response' ] == 'hidd3n_valu3'
			&& $_SERVER[ 'HTTP_USER_AGENT' ] == 'reCAPTCHA'
		)
	){
		// CAPTCHA was correct. Do both new passwords match?
		if ($pass_new == $pass_conf) {
			$pass_new = ((isset($GLOBALS["___mysqli_ston"]) && is_object($GLOBALS["___mysqli_ston"])) ? mysqli_real_escape_string($GLOBALS["___mysqli_ston"],  $pass_new ) : ((trigger_error("[MySQLConverterToo] Fix the mysql_escape_string() call! This code does not work.", E_USER_ERROR)) ? "" : ""));
			$pass_new = md5( $pass_new );

			// Update database
			$insert = "UPDATE `users` SET password = '$pass_new' WHERE user = '" . dvwaCurrentUser() . "' LIMIT 1;";
			$result = mysqli_query($GLOBALS["___mysqli_ston"],  $insert ) or die( '<pre>' . ((is_object($GLOBALS["___mysqli_ston"])) ? mysqli_error($GLOBALS["___mysqli_ston"]) : (($___mysqli_res = mysqli_connect_error()) ? $___mysqli_res : false)) . '</pre>' );

			// Feedback for user
			$html .= "<pre>Password Changed.</pre>";

		} else {
			// Ops. Password mismatch
			$html     .= "<pre>Both passwords must match.</pre>";
			$hide_form = false;
		}

	} else {
		// What happens when the CAPTCHA was entered incorrectly
		$html     .= "<pre><br />The CAPTCHA was incorrect. Please try again.</pre>";
		$hide_form = false;
		return;
	}

	((is_null($___mysqli_res = mysqli_close($GLOBALS["___mysqli_ston"]))) ? false : $___mysqli_res);
}

// Generate Anti-CSRF token
generateSessionToken();

?>
```

### 漏洞成因

这里的漏洞点其实已经不是验证和修改分离了，而是开发者预留的后门被利用了。

```php
if (
		$resp || 
		(
			$_POST[ 'g-recaptcha-response' ] == 'hidd3n_valu3'
			&& $_SERVER[ 'HTTP_USER_AGENT' ] == 'reCAPTCHA'
		)
	)
```

这段代码说明：如果验证码正确或者请求体中有`g-recaptcha-response=hidd3n_valu3`且User-Agent=reCAPTCHA时会进行密码修改。攻击者可以利用这个后门实现密码修改。

### 利用手法

抓包之后请求体中添加g-recaptcha-response=hidd3n_valu3，并设置UA头为reCAPTCHA。

## 等级impossible

### 源码

```php
<?php

if( isset( $_POST[ 'Change' ] ) ) {
	// Check Anti-CSRF token
	checkToken( $_REQUEST[ 'user_token' ], $_SESSION[ 'session_token' ], 'index.php' );

	// Hide the CAPTCHA form
	$hide_form = true;

	// Get input
	$pass_new  = $_POST[ 'password_new' ];
	$pass_new  = stripslashes( $pass_new );
	$pass_new  = ((isset($GLOBALS["___mysqli_ston"]) && is_object($GLOBALS["___mysqli_ston"])) ? mysqli_real_escape_string($GLOBALS["___mysqli_ston"],  $pass_new ) : ((trigger_error("[MySQLConverterToo] Fix the mysql_escape_string() call! This code does not work.", E_USER_ERROR)) ? "" : ""));
	$pass_new  = md5( $pass_new );

	$pass_conf = $_POST[ 'password_conf' ];
	$pass_conf = stripslashes( $pass_conf );
	$pass_conf = ((isset($GLOBALS["___mysqli_ston"]) && is_object($GLOBALS["___mysqli_ston"])) ? mysqli_real_escape_string($GLOBALS["___mysqli_ston"],  $pass_conf ) : ((trigger_error("[MySQLConverterToo] Fix the mysql_escape_string() call! This code does not work.", E_USER_ERROR)) ? "" : ""));
	$pass_conf = md5( $pass_conf );

	$pass_curr = $_POST[ 'password_current' ];
	$pass_curr = stripslashes( $pass_curr );
	$pass_curr = ((isset($GLOBALS["___mysqli_ston"]) && is_object($GLOBALS["___mysqli_ston"])) ? mysqli_real_escape_string($GLOBALS["___mysqli_ston"],  $pass_curr ) : ((trigger_error("[MySQLConverterToo] Fix the mysql_escape_string() call! This code does not work.", E_USER_ERROR)) ? "" : ""));
	$pass_curr = md5( $pass_curr );

	// Check CAPTCHA from 3rd party
	$resp = recaptcha_check_answer(
		$_DVWA[ 'recaptcha_private_key' ],
		$_POST['g-recaptcha-response']
	);

	// Did the CAPTCHA fail?
	if( !$resp ) {
		// What happens when the CAPTCHA was entered incorrectly
		$html .= "<pre><br />The CAPTCHA was incorrect. Please try again.</pre>";
		$hide_form = false;
	}
	else {
		// Check that the current password is correct
		$data = $db->prepare( 'SELECT password FROM users WHERE user = (:user) AND password = (:password) LIMIT 1;' );
		$data->bindParam( ':user', dvwaCurrentUser(), PDO::PARAM_STR );
		$data->bindParam( ':password', $pass_curr, PDO::PARAM_STR );
		$data->execute();

		// Do both new password match and was the current password correct?
		if( ( $pass_new == $pass_conf) && ( $data->rowCount() == 1 ) ) {
			// Update the database
			$data = $db->prepare( 'UPDATE users SET password = (:password) WHERE user = (:user);' );
			$data->bindParam( ':password', $pass_new, PDO::PARAM_STR );
			$data->bindParam( ':user', dvwaCurrentUser(), PDO::PARAM_STR );
			$data->execute();

			// Feedback for the end user - success!
			$html .= "<pre>Password Changed.</pre>";
		}
		else {
			// Feedback for the end user - failed!
			$html .= "<pre>Either your current password is incorrect or the new passwords did not match.<br />Please try again.</pre>";
			$hide_form = false;
		}
	}
}

// Generate Anti-CSRF token
generateSessionToken();

?>
```

在impossible的代码中，没有设置会被利用的后门，用参数化查询防御SQL注入，验证和修改的步骤也没有分离，验证码因此无法绕过，同时加强了身份认证。

## 总结

在web开发时，涉及到验证码问题，要把验证码和修改密码的操作结合，同时减少后门，还要注意其他漏洞点的存在。在漏洞利用时，可以看看网页源码来猜测后端是怎么进行验证码校验和修改操作的。