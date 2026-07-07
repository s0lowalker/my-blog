---
title: JD2M源码分析
date: 2026-07-07
---

# JD2M源码分析

## Unicode2CN.java

这个文件的核心功能是把 Java 源码里的 \uXXXX Unicode 转义序列还原成可读的中文字符

CFR 反编译字节码时，遇到中文字符串会输出成 \u4e2d\u6587 这种形态。JD2M 用这个类做后处理，把反编译结果还原成人类可读的 中文。

### 方法功能

#### 1. ascii2Char(String str) — 最底层：单个 \uXXXX 转 char

```java
private static char ascii2Char(String str) {
	if (str.length() != 6) {
		throw new IllegalArgumentException("Length must be 6, got: " + str.length());    //\uXXXX必须是6个字符
	}
	if (!"\\u".equals(str.substring(0, 2))) {    //前缀检查
		throw new IllegalArgumentException("Must start with \\u");
	}
	String tmp = str.substring(2, 4);
	int code = Integer.parseInt(tmp, 16) << 8;
	tmp = str.substring(4, 6);
	code += Integer.parseInt(tmp, 16);
	return (char) code;
}
```

由于 Java 字符串里 `\u` 要写成 `\\u`，因为 `\` 本身需要转义。所以传进来的参数长这样：`\\u4e16`（6 个字符：`\ u 4 e 1 6`）。

`String tmp = str.substring(2, 4)`截取到 unicode 字符的2到4位，`int code = Integer.parseInt(tmp, 16) << 8`把截取到的`tmp`当作16进制解析并转化为十进制的`int`整数然后再左移8位，以便拼接后续8位。

#### 2. isHex(String str, int start, int end) — 辅助：校验是不是十六进制

```java
private static boolean isHex(String str, int start, int end) {
    for (int i = start; i < end; i++) {
        char c = str.charAt(i);
        if (!((c >= '0' && c <= '9') || (c >= 'a' && c <= 'f') || (c >= 'A' && c <= 'F'))) {
            return false;
        }
    }
    return true;
}
```

对整个字符串遍历来判断是不是16进制。

#### 3. ascii2Native(String str)

```java
public static String ascii2Native(String str) {
	StringBuilder sb = new StringBuilder();
	int begin = 0;    //整个字符串的起始位置
	int index = str.indexOf("\\u");    //获取\u出现的位置，如果不存在则返回-1
	while (index != -1) {
		sb.append(str, begin, index);  //把起始位置和\u之间的字符串放到结果中
		if (index + 6 <= str.length() && isHex(str, index + 2, index + 6)) {
			sb.append(ascii2Char(str.substring(index, index + 6)));//转化为中文
			begin = index + 6;//起始位置后移
		} else {     //后续不是unicode字符
			sb.append("\\u");
			begin = index + 2;
		}
		index = str.indexOf("\\u", begin);    //从起始位置开始找下一个出现\\u的位置
	}
	sb.append(str.substring(begin));   //把从起始位置到末尾的所有字符加到结果中
	return sb.toString();  //把结果转化为字符串返回
}
```

这个方法的作用是：**把字符串中所有 `\uXXXX` 格式的 Unicode 转义序列，替换成对应的真实字符。**

`StringBuilder` 是 Java 中用于**高效拼接字符串**的工具类。Java 的 `String` 是**不可变的**，每次拼接都会创建新对象，如果循环里频繁拼接，会产生大量临时对象，**效率低、浪费内存**。

#### 4. toCN(File file, File aim)

```java
private static void toCN(File file, File aim) throws IOException { //源文件与目标文件
    List<String> allLines = new ArrayList<>(1024); //创建列表存储所有行，假设有1024行
    //使用hutool这个java工具库的方法以UFT8编码读取源文件的所有行并存入列表
    FileUtil.readLines(file, StandardCharsets.UTF_8, allLines);
    for (int i = 0; i < allLines.size(); i++) {     //遍历所有行
        //allLines.get(i)获取第i行的原始内容
        //ascii2Native(allLines.get(i))把第i行的unicode转化为中文
        //allLines.set(i, ...)用转化后的内容替换原来的行
        allLines.set(i, ascii2Native(allLines.get(i)));
    }
    //把转化后的列表中的所有行写入目标文件，以UTF8编码的形式
    FileUtil.writeLines(allLines, aim, StandardCharsets.UTF_8);
}
```

这个方法的作用是：**读取一个文件，把里面所有行的 `\uXXXX` 转义序列都转换成真实的中文字符，然后写入到另一个文件。**

#### 5. toCN(File sourceFile)

```java
public static void toCN(File sourceFile) throws IOException {
    //创建临时文件，在源文件路径后加.tmp
	File aimFile = new File(sourceFile.getPath() + ".tmp");
    //调用重载的方法
	toCN(sourceFile, aimFile);
    //FileUtil.move是hutool的文件移动方法
    //第三个参数true表示如果目标文件存在就覆盖它
    //把.tmp文件移动（重命名）为源文件的名字
	FileUtil.move(aimFile, sourceFile, true);
}
```

这个方法的作用是：**把源文件中的 `\uXXXX` 转成中文后，直接替换掉源文件。**这个方法就是**封装了"转换后覆盖原文件"的逻辑**。

## StringExtractor.java

这个文件是一个 Java 字节码字符串提取工具。核心作用是把从编译好的`.class`文件、`.jar`/`.war`包或目录中提取出所有长度超过指定值的字符串，然后打印出来。

这个工具底层使用 ASM 字节码框架，Java 编译后的 `.class` 文件是一堆二进制字节，ASM 库可以**读取并解析这些字节**，然后通过**访问者模式**把字节码的各个部分"汇报"给一个访问者对象。

汇报的内容包括：

- 类名、父类名、接口名
- 字段名、字段类型
- 方法名、方法签名
- 方法体里的字符串常量
- 注解信息
- 等等

**核心机制**：`ClassReader` 负责读取字节码，`ClassVisitor` 负责接收汇报。我们只需要写一个 `ClassVisitor` 的子类，在对应的方法里收集信息即可。

### StringCollector 内部类 - 自定义访问者

```java
private static class StringCollector extends ClassVisitor {
    final Set<String> strings = new LinkedHashSet<>();
    final int minLength;

    StringCollector(int minLength) {
        super(Opcodes.ASM9);
        this.minLength = minLength;
    }

    private void add(String s) {
        if (s != null && s.length() >= minLength) {
            strings.add(s);
        }
    }

    @Override
    public void visit(int version, int access, String name, String signature,
                      String superName, String[] interfaces) {
        add(name);
        add(superName);
        if (interfaces != null) {
            for (String iface : interfaces) add(iface);
        }
        super.visit(version, access, name, signature, superName, interfaces);
    }

    @Override
    public FieldVisitor visitField(int access, String name, String descriptor,
                                    String signature, Object value) {
        add(name);
        add(descriptor);
        if (value instanceof String) add((String) value);
        return super.visitField(access, name, descriptor, signature, value);
    }

    @Override
    public MethodVisitor visitMethod(int access, String name, String descriptor,
                                      String signature, String[] exceptions) {
        add(name);
        add(descriptor);
        if (exceptions != null) {
            for (String ex : exceptions) add(ex);
        }
        return new MethodVisitor(Opcodes.ASM9) {
            @Override
            public void visitLdcInsn(Object cst) {
                if (cst instanceof String) add((String) cst);
                super.visitLdcInsn(cst);
            }

            @Override
            public void visitInvokeDynamicInsn(String name, String descriptor,
                                                Handle bootstrapMethodHandle,
                                                Object... bootstrapMethodArguments) {
                add(name);
                for (Object arg : bootstrapMethodArguments) {
                    if (arg instanceof String) add((String) arg);
                    if (arg instanceof Handle) {
                        Handle h = (Handle) arg;
                        add(h.getName());
                        add(h.getDesc());
                        add(h.getOwner());
                    }
                }
                super.visitInvokeDynamicInsn(name, descriptor, bootstrapMethodHandle, bootstrapMethodArguments);
            }
        };
    }

    @Override
    public AnnotationVisitor visitAnnotation(String descriptor, boolean visible) {
        add(descriptor);
        return new AnnotationCollector();
    }

    @Override
    public AnnotationVisitor visitTypeAnnotation(int typeRef, TypePath typePath,
                                                  String descriptor, boolean visible) {
        add(descriptor);
        return new AnnotationCollector();
    }

    private class AnnotationCollector extends AnnotationVisitor {
        AnnotationCollector() {
            super(Opcodes.ASM9);
        }

        @Override
        public void visit(String name, Object value) {
            add(name);
            if (value instanceof String) add((String) value);
            super.visit(name, value);
        }

        @Override
        public void visitEnum(String name, String descriptor, String value) {
            add(name);
            add(descriptor);
            add(value);
            super.visitEnum(name, descriptor, value);
        }

        @Override
        public AnnotationVisitor visitAnnotation(String name, String descriptor) {
            add(name);
            add(descriptor);
            return new AnnotationCollector();
        }

        @Override
        public AnnotationVisitor visitArray(String name) {
            add(name);
            return new AnnotationCollector();
        }
    }
}
```

这是整个工具的核心，一个自定义的`ClassVisitor`，重写了各种回调方法。

#### 数据结构

```java
final Set<String> strings = new LinkedHashSet<>();
```

`LinkedHashSet`这个数据结构有三个特性：去重、保持插入顺序、O(1)的查找和插入。

这就保证了同一个字符串只需收集一次、元素的顺序不变收集字符串的速度很快。

#### add()

```java
private void add(String s) {
    if (s != null && s.length() >= minLength) {
        strings.add(s);
    }
}
```

这个方法是整个`StringCollector`的**唯一入口**，所有收集到的字符串都要经过它。

由于字节码中有大量的**短字符串**，比如`get`之类的方法名，方法描述符、字段名等。我们要找的主要是像“操作成功”这样的较长的字符串。然后把收集到的字符串给添加到`strings`这个`LinkedHashSet`。

#### visit() - 访问类的基本信息

```java
@Override
public void visit(int version, int access, String name, String signature,
                  String superName, String[] interfaces) {
    add(name);//name是ASM内部格式的类名，用/分隔，如com/example/UserService。
    add(superName);//收集父类名
    if (interfaces != null) {   //收集接口名
        for (String iface : interfaces) add(iface);
    }
    //这一行非常重要，把调用传递给父类ClassVisitor，让ASM继续解析这个类的内部
    super.visit(version, access, name, signature, superName, interfaces);
}
```

这是 `ClassVisitor.visit()` 方法的重写，**在 ASM 解析到类的定义头时被回调**。当`ClassReader.accept()`扫描到类定义的开头部分，也就是在代码中像`public class UserServiceImpl extends BaseService implements Serializable {`这一行时就会自动调用`ClassVisitor.visit()`。

| 参数         | 含义           | 示例                        |
| :----------- | :------------- | :-------------------------- |
| `version`    | 字节码版本号   | 52（Java 8），61（Java 17） |
| `access`     | 访问修饰符     | `ACC_PUBLIC + ACC_SUPER`    |
| `name`       | 类的内部名称   | `org/example/Hello`         |
| `signature`  | 泛型签名       | `Lorg/example/Hello<TT;>;`  |
| `superName`  | 父类内部名称   | `java/lang/Object`          |
| `interfaces` | 实现的接口列表 | `["java/io/Serializable"]`  |

假设有这样的类：

```java
package com.example;

public class UserServiceImpl extends BaseService implements Serializable, Comparable<User> {
    // ...
}
```

回调时参数值为：

```text
name:       "com/example/UserServiceImpl"
superName:  "com/example/BaseService"
interfaces: ["java/io/Serializable", "java/lang/Comparable"]
```

会尝试收集：

| 字符串                        | 长度 | 是否收集 |
| :---------------------------- | :--- | :------- |
| `com/example/UserServiceImpl` | 29   | ✅        |
| `com/example/BaseService`     | 22   | ✅        |
| `java/io/Serializable`        | 20   | ✅        |
| `java/lang/Comparable`        | 20   | ✅        |

这个方法的逻辑非常简单：

```text
visit() 被回调
  │
  ├── 收集类名 (name)
  ├── 收集父类名 (superName)
  ├── 遍历收集所有接口名 (interfaces[])
  │
  └── super.visit(...) → 让 ASM 继续解析类内部
```

它确保**类的基本身份信息**（类名、父类、接口）都能被提取，为后续深入类内部打开通道。

#### visitField() - 访问字段

```java
@Override
public FieldVisitor visitField(int access, String name, String descriptor,
                                String signature, Object value) {
    add(name);  //收集字段名
    add(descriptor); //收集类型描述符
    if (value instanceof String) add((String) value);  //收集静态常量字段的值
    //只有当value是静态的、常量的、基本类型或String类型时才不是null
    //返回一个FieldVisitor对象，ASM拿到这个对象后会用它来解析这个字段上的注解
    return super.visitField(access, name, descriptor, signature, value);
}
```

这是 `ClassVisitor.visitField()` 方法的重写，**在 ASM 解析到类中的一个字段时被回调**。

| 参数         | 含义                 | 示例                                     |
| :----------- | :------------------- | :--------------------------------------- |
| `access`     | 访问修饰符           | `ACC_PRIVATE + ACC_STATIC + ACC_FINAL`   |
| `name`       | 字段名               | `"userName"`, `"MSG"`                    |
| `descriptor` | 类型描述符           | `"Ljava/lang/String;"`, `"I"`            |
| `signature`  | 泛型签名             | `"Ljava/util/List<Ljava/lang/String;>;"` |
| `value`      | 静态常量字段的初始值 | `"操作成功"`，非静态字段为 `null`        |

#### visitMethod() - 访问方法

```java
@Override
public MethodVisitor visitMethod(int access, String name, String descriptor,
                                  String signature, String[] exceptions) {
    add(name);   //收集方法名
    add(descriptor);   //收集方法描述符
    if (exceptions != null) {     
        for (String ex : exceptions) add(ex);   //收集异常类型
    }
    return new MethodVisitor(Opcodes.ASM9) {   
        //返回MethodVisitor来深入方法体并按照 ASM 9 的规范来处理方法体
        @Override      //收集字符串常量 ldc：将字符串加载到操作数栈
        public void visitLdcInsn(Object cst) {   
            //java代码中的每一个字符串字面量都会在字节码中以LDC指令出现
            //每次ASM在方法体内遇到LDC指令就会回调这个方法，把常量传进来
            if (cst instanceof String) add((String) cst);
            super.visitLdcInsn(cst);   
        }

        @Override   //收集动态调用中的字符串，ASM在方法体内遇到invokedynamic指令时回调
        public void visitInvokeDynamicInsn(String name, String descriptor,
                                            Handle bootstrapMethodHandle,
                                            Object... bootstrapMethodArguments) {
            //name：调用点名称 descriptor：方法描述符 bootstrapMethodHandle：引导方法的句柄
            //bootstrapMethodArguments：传给引导方法的额外参数
            add(name);
            for (Object arg : bootstrapMethodArguments) {
                if (arg instanceof String) add((String) arg);
                if (arg instanceof Handle) { //拆解句柄的三个部分
                    Handle h = (Handle) arg;
                    add(h.getName());//方法名
                    add(h.getDesc());//方法描述符
                    add(h.getOwner());//方法所在类
                }
            }
            super.visitInvokeDynamicInsn(name, descriptor, bootstrapMethodHandle, bootstrapMethodArguments);
        }
    };
}
```

这个函数做了两件事：**收集方法签名信息** + **返回一个匿名 MethodVisitor 深入方法体内部**。

| 参数         | 含义                          | 示例                             |
| :----------- | :---------------------------- | :------------------------------- |
| `access`     | 访问修饰符                    | `ACC_PUBLIC + ACC_STATIC`        |
| `name`       | 方法名                        | `"sayHello"`, `"main"`           |
| `descriptor` | 方法描述符（参数+返回值类型） | `"(Ljava/lang/String;)V"`        |
| `signature`  | 泛型签名                      | `"<T:Ljava/lang/Object;>(TT;)V"` |
| `exceptions` | 声明的异常类型列表            | `["java/io/IOException"]`        |

#### AnnotationCollector - 递归收集注解内所有字符串

```java
private class AnnotationCollector extends AnnotationVisitor {
    AnnotationCollector() {
        super(Opcodes.ASM9); //告诉父类按照ASM9规范解析
    }

    @Override
    public void visit(String name, Object value) {  //处理键值对
        add(name); //收集键名
        if (value instanceof String) add((String) value); //收集字符串值
        super.visit(name, value);
    }

    @Override   //处理枚举值
    public void visitEnum(String name, String descriptor, String value) {
        add(name);  //收集键名
        add(descriptor); //枚举类型描述符
        add(value); //枚举值的名字
        super.visitEnum(name, descriptor, value);
    }

    @Override   //处理嵌套注解
    public AnnotationVisitor visitAnnotation(String name, String descriptor) {
        add(name);  //键名
        add(descriptor);//嵌套注解描述符
        return new AnnotationCollector(); //递归处理注解
    }

    @Override   //处理数组
    public AnnotationVisitor visitArray(String name) {
        add(name);  //数组键名
        return new AnnotationCollector();//返回新收集器，处理数组每个元素
    }
}
```

`AnnotationCollector` 继承自 `AnnotationVisitor`，专门处理**注解内部**的各种元素。当 `visitAnnotation()` 或 `visitTypeAnnotation()` 被调用后，ASM 需要继续深入注解内部，就是这个类负责接待。

一个注解内部可以包含：

```java
@Example(
    name = "user",                    // ① 键值对 (String)
    count = 10,                       // ① 键值对 (int)
    type = MyEnum.A,                  // ② 枚举值
    nested = @Nested(value = "x"),    // ③ 嵌套注解
    tags = {"a", "b"}                 // ④ 数组
)
```

| 元素类型 | 对应回调方法                   | 可能包含字符串的位置                     |
| :------- | :----------------------------- | :--------------------------------------- |
| 键值对   | `visit(name, value)`           | 键名、String 类型的值                    |
| 枚举值   | `visitEnum(name, desc, value)` | 键名、枚举描述符、枚举值名               |
| 嵌套注解 | `visitAnnotation(name, desc)`  | 键名、嵌套注解的描述符，以及**递归进去** |
| 数组     | `visitArray(name)`             | 键名，以及**数组内每个元素递归进去**     |

#### visitAnnotation() - 访问注解

```java
public AnnotationVisitor visitAnnotation(String descriptor, boolean visible) {
    add(descriptor); //收集注解类型描述符，只收集注解的类型名，但不收集注解里的键值对
    return new AnnotationCollector();
    //返回AnnotationVisitor，ASM拿到这个对象之后会用它继续解析注解内部的键值对
}
```

这是 `ClassVisitor.visitAnnotation()` 方法的重写，**在 ASM 解析到类上的一个注解时被回调**。

| 参数         | 含义             | 示例                                         |
| :----------- | :--------------- | :------------------------------------------- |
| `descriptor` | 注解的类型描述符 | `"Lorg/springframework/stereotype/Service;"` |
| `visible`    | 运行时是否可见   | `true`（运行时注解）/ `false`（编译时注解）  |

#### visitTypeAnnotation() - 访问类型注解

```java
@Override
public AnnotationVisitor visitTypeAnnotation(int typeRef, TypePath typePath,
                                              String descriptor, boolean visible) {
    add(descriptor);
    return new AnnotationCollector();
}
```

类型注解是 **Java 8 引入**的新特性。和普通注解不同，它可以贴在**任何使用类型的地方**：

```java
// 普通注解：只能贴在声明上
@Override
public String toString() { ... }

// 类型注解：可以贴在任何类型上
@NotNull String name;                           // 字段类型
List<@NonNull String> list;                      // 泛型参数
public @Nullable String getName() { ... }        // 返回值类型
void foo(@NotNull Object obj) { ... }            // 方法参数类型
new @Interned MyObject();                        // 对象创建
```

| 参数         | 含义             | 示例                                             |
| :----------- | :--------------- | :----------------------------------------------- |
| `typeRef`    | 目标类型引用     | 表示贴在字段上、方法返回值上、参数上等           |
| `typePath`   | 复杂类型中的路径 | 如 `List<@NotNull String>` 中，路径指向 `String` |
| `descriptor` | 注解描述符       | `"Ljavax/validation/constraints/NotNull;"`       |
| `visible`    | 运行时可见性     | `true`（运行时保留）/ `false`（仅编译时）        |

### 三个入口方法

#### extractFromClass(File, minLength) - 单个class文件提取字符串

```java
public static List<String> extractFromClass(File classFile, int minLength) throws IOException {
    byte[] bytes = FileUtil.readBytes(classFile);//hutool的工具方法，读取字节码
    ClassReader cr = new ClassReader(bytes); //交给ASM处理，ASM已经知道字节码的一切，等待汇报
    StringCollector collector = new StringCollector(minLength);//创建字符串收集器
    cr.accept(collector, ClassReader.SKIP_DEBUG | ClassReader.SKIP_FRAMES);
    //启动ASM的遍历，遇到任何东西都告诉收集器，依次回调相关方法，会跳过不必要的信息
    return new ArrayList<>(collector.strings);//把LinkedHashSet<String>包装一下方便调用
}
```

#### extractFromJar(File, minLength) - jar/war包提取字符串

```java
public static Map<String, List<String>> extractFromJar(File jarFile, int minLength) throws IOException {
    Map<String, List<String>> result = new LinkedHashMap<>();
    JarFile jar = new JarFile(jarFile);//读取jar/zip格式的文件
    try {
        Enumeration<JarEntry> entries = jar.entries();//返回jar包内所有文件和目录的枚举
        while (entries.hasMoreElements()) {
            //逐个取出每个JarEntry代表一个文件或目录
            JarEntry entry = entries.nextElement();
            //跳过目录和非class文件
            if (entry.isDirectory() || !entry.getName().endsWith(".class")) continue;
            //读取并解析单个class文件
            try (InputStream is = jar.getInputStream(entry)) {
                ClassReader cr = new ClassReader(is);
                StringCollector collector = new StringCollector(minLength);
                cr.accept(collector, ClassReader.SKIP_DEBUG | ClassReader.SKIP_FRAMES);
                //如果提取到字符串，存入结果
                if (!collector.strings.isEmpty()) {
                    //换成java类名格式
                    String className = entry.getName()
                            .replace('/', '.')
                            .replace(".class", "");
                    result.put(className, new ArrayList<>(collector.strings));
                }
            } catch (Exception ignored) {
            }
        }
    } finally {
        jar.close(); //关闭jar包
    }
    return result;
}
```

| 参数        | 含义                                |
| :---------- | :---------------------------------- |
| `jarFile`   | `.jar` 或 `.war` 文件               |
| `minLength` | 最小字符串长度过滤                  |
| 返回值      | `Map<类名, 该类提取到的字符串列表>` |

#### extractFromDir(File, minLength)

```java
public static Map<String, List<String>> extractFromDir(File dir, int minLength) throws IOException {
    Map<String, List<String>> result = new LinkedHashMap<>();//创建结果容器
    collectAndExtract(dir, dir, result, minLength);//调用递归方法
    return result;
}
//递归核心
private static void collectAndExtract(File root, File current,
                                       Map<String, List<String>> result, int minLength) {
    File[] files = current.listFiles();//列出当前目录下的所有文件
    if (files == null) return;//没有访问权限、路径不是目录等
    //遍历每一个文件/目录
    for (File f : files) {
        if (f.isDirectory()) {
            collectAndExtract(root, f, result, minLength);//是目录就递归调用
        } else if (f.getName().endsWith(".class")) { //遇到class文件就提取
            try {
                List<String> strings = extractFromClass(f, minLength);//单文件提取
                if (!strings.isEmpty()) {
                    //计算相对路径
                    String relPath = f.getAbsolutePath()
                            .substring(root.getAbsolutePath().length() + 1)
                            .replace(File.separator, ".")
                            .replace(".class", "");
                    result.put(relPath, strings);
                }
            } catch (IOException ignored) {
            }
        }
    }
}
```

**参数说明**

| 参数        | 含义               | 作用                           |
| :---------- | :----------------- | :----------------------------- |
| `root`      | 根目录             | **固定不变**，用于计算相对路径 |
| `current`   | 当前正在遍历的目录 | **每次递归变化**，逐层深入     |
| `result`    | 结果容器           | 所有递归层级共享同一个 Map     |
| `minLength` | 最小长度           | 透传给 `extractFromClass`      |

### 方法编排

#### extractFromInput() - 入口

```java
public static void extractFromInput(File input, int minLength) {
    try {
        long start = System.currentTimeMillis();//记录开始时间，最后计算耗时
        System.out.println("[JD2M] Extracting strings from: " + input.getAbsolutePath());   //输出正在处理的文件/目录 
        System.out.println("[JD2M] Minimum length: " + minLength);//输出过滤阈值
        System.out.println();
		//单个class文件
        if (input.isFile() && input.getName().toLowerCase().endsWith(".class")) {
            List<String> strings = extractFromClass(input, minLength);
            System.out.println("--- " + input.getName() + " ---");
            for (String s : strings) {
                System.out.println("  " + s);
            }
            System.out.println();
            System.out.println("Total: " + strings.size() + " string(s)");
			//jar/war包
        } else if (input.isFile() &&
                (input.getName().toLowerCase().endsWith(".jar") ||
                 input.getName().toLowerCase().endsWith(".war"))) {
            Map<String, List<String>> result = extractFromJar(input, minLength);
            int total = printResults(result);
            System.out.println("Total: " + total + " string(s) in " + result.size() + " class(es)");

        } else if (input.isDirectory()) {
            Map<String, List<String>> result = extractFromDir(input, minLength);
            int total = printResults(result);
            System.out.println("Total: " + total + " string(s) in " + result.size() + " class(es)");
			//目录
        } else {
            System.err.println("[JD2M] Unsupported input: " + input.getName());
            return;
        }

        long elapsed = System.currentTimeMillis() - start;
        System.out.println("[JD2M] Done in " + (elapsed / 1000) + "s");

    } catch (IOException e) {
        System.err.println("[JD2M] Error: " + e.getMessage());
        e.printStackTrace();
    }
}
```

这是整个工具的入口，负责判断输入类型、分发到对应的处理方法、打印结果。

#### printResults() - 打印结果

```java
private static int printResults(Map<String, List<String>> result) {
    int total = 0;   //统计字符串总数
    for (Map.Entry<String, List<String>> entry : result.entrySet()) {
        System.out.println("--- " + entry.getKey() + " ---");
        for (String s : entry.getValue()) {
            System.out.println("  " + s);
        }
        total += entry.getValue().size();//计数
    }
    System.out.println();
    return total;
}
```

## Decompiler.java

```java
package org.jd2m;

import cn.hutool.core.io.FileUtil;
import org.benf.cfr.reader.api.CfrDriver;
import org.benf.cfr.reader.util.getopt.OptionsImpl;
import org.objectweb.asm.ClassReader;
import org.objectweb.asm.ClassVisitor;
import org.objectweb.asm.Opcodes;

import java.io.File;
import java.io.IOException;
import java.io.InputStream;
import java.nio.file.Files;
import java.nio.file.Path;
import java.util.*;
import java.util.concurrent.*;
import java.util.concurrent.atomic.AtomicInteger;
import java.util.jar.JarEntry;
import java.util.jar.JarFile;

public class Decompiler {

    private static final String[] JAR_EXTENSIONS = {".jar", ".war", ".ear"};
    private static final String[] SKIP_PREFIXES = {"META-INF", "WEB-INF"};
    private static final String BOOT_INF = "BOOT-INF/";
    private static final String BOOT_INF_CLASSES = "BOOT-INF/classes/";

    private static final ExecutorService executor = new ThreadPoolExecutor(
            8, 8, 60, TimeUnit.SECONDS,
            new LinkedBlockingQueue<>(),
            new ThreadPoolExecutor.CallerRunsPolicy());

    public static Result decompile(File input, File output, boolean mavenExport) throws IOException {
        if (!input.exists()) {
            throw new IllegalArgumentException("Input not found: " + input.getAbsolutePath());
        }

        long start = System.currentTimeMillis();
        System.out.println("[JD2M] Input : " + input.getAbsolutePath());
        System.out.println("[JD2M] Output: " + output.getAbsolutePath());

        Result result;
        if (input.isFile()) {
            result = decompileFile(input, output, mavenExport);
        } else {
            result = decompileDirectory(input, output, mavenExport);
        }

        convertUnicode(output);
        executor.shutdown();

        long elapsed = System.currentTimeMillis() - start;
        System.out.println("[JD2M] Done in " + (elapsed / 1000) + "s — " +
                result.success + " OK, " + result.failed + " failed");

        if (!result.failedClasses.isEmpty()) {
            System.out.println("[JD2M] Failed classes (" + result.failedClasses.size() + "):");
            for (String cls : result.failedClasses) {
                System.out.println("  FAIL: " + cls);
            }
        }
        return result;
    }

    private static Result decompileFile(File file, File output, boolean mavenExport) throws IOException {
        String name = file.getName().toLowerCase();

        if (name.endsWith(".class")) {
            String javaOutput = output.getAbsolutePath();
            if (mavenExport) {
                javaOutput = output.getAbsolutePath() + File.separator + "src" +
                        File.separator + "main" + File.separator + "java";
                generatePomForClassFile(file, output);
            }
            return decompileSingleClass(file.getAbsolutePath(), javaOutput);
        }

        for (String ext : JAR_EXTENSIONS) {
            if (name.endsWith(ext)) {
                String javaOutput = output.getAbsolutePath();
                if (mavenExport) {
                    javaOutput = output.getAbsolutePath() + File.separator + "src" +
                            File.separator + "main" + File.separator + "java";
                }

                Result result = decompileJarClassByClass(file, new File(javaOutput));

                if (mavenExport) {
                    extractMavenResources(file, output);
                }
                return result;
            }
        }

        throw new IllegalArgumentException("Unsupported file type: " + file.getName());
    }

    private static Result decompileDirectory(File dir, File output, boolean mavenExport) {
        List<String> classFiles = new ArrayList<>();
        collectClassFiles(dir, classFiles);

        if (classFiles.isEmpty()) {
            System.out.println("[JD2M] No .class files found.");
            return new Result(0, 0, Collections.emptyList());
        }

        System.out.println("[JD2M] Found " + classFiles.size() + " .class file(s), decompiling one by one...");
        String javaOutput = output.getAbsolutePath();
        if (mavenExport) {
            javaOutput = output.getAbsolutePath() + File.separator + "src" +
                    File.separator + "main" + File.separator + "java";
            generatePomForDir(dir, output);
        }
        return decompileClassList(classFiles, javaOutput);
    }

    private static void collectClassFiles(File dir, List<String> result) {
        File[] files = dir.listFiles();
        if (files == null) return;
        for (File f : files) {
            if (f.isDirectory()) {
                collectClassFiles(f, result);
            } else if (f.getName().endsWith(".class")) {
                result.add(f.getAbsolutePath());
            }
        }
    }

    private static Result decompileJarClassByClass(File jarFile, File outputDir) {
        Path tempDir = null;
        try {
            tempDir = Files.createTempDirectory("jd2m-");
            List<String> classPaths = new ArrayList<>();

            JarFile jar = new JarFile(jarFile);
            try {
                Enumeration<JarEntry> entries = jar.entries();
                while (entries.hasMoreElements()) {
                    JarEntry entry = entries.nextElement();
                    if (entry.isDirectory() || !entry.getName().endsWith(".class")) continue;

                    File destFile = new File(tempDir.toFile(), entry.getName());
                    destFile.getParentFile().mkdirs();
                    try (InputStream is = jar.getInputStream(entry)) {
                        FileUtil.writeFromStream(is, destFile);
                    }
                    classPaths.add(destFile.getAbsolutePath());
                }
            } finally {
                jar.close();
            }

            System.out.println("[JD2M] Extracted " + classPaths.size() + " class(es), decompiling...");

            Result result = decompileClassList(classPaths, outputDir.getAbsolutePath());
            return result;

        } catch (IOException e) {
            return new Result(0, 0, Collections.singletonList("jar-extract: " + e.getMessage()));
        } finally {
            if (tempDir != null) {
                FileUtil.del(tempDir.toFile());
            }
        }
    }

    private static Result decompileClassList(List<String> classPaths, String outputPath) {
        int success = 0;
        int failed = 0;
        List<String> failedClasses = new ArrayList<>();
        AtomicInteger progress = new AtomicInteger(0);
        int total = classPaths.size();

        for (String classPath : classPaths) {
            try {
                doCfrDecompile(Collections.singletonList(classPath), outputPath);
                success++;
            } catch (Exception e) {
                failed++;
                String shortName = shortenPath(classPath);
                failedClasses.add(shortName);
                System.err.println("[JD2M] Decompile failed: " + shortName + " — " + e.getMessage());
            }

            int done = progress.incrementAndGet();
            if (total > 20 && done % (total / 10) == 0) {
                System.out.println("[JD2M] Progress: " + done + "/" + total);
            }
        }
        return new Result(success, failed, failedClasses);
    }

    private static Result decompileSingleClass(String classPath, String outputPath) {
        try {
            doCfrDecompile(Collections.singletonList(classPath), outputPath);
            return new Result(1, 0, Collections.emptyList());
        } catch (Exception e) {
            System.err.println("[JD2M] Decompile failed: " + shortenPath(classPath) + " — " + e.getMessage());
            return new Result(0, 1, Collections.singletonList(shortenPath(classPath)));
        }
    }

    private static void doCfrDecompile(List<String> sourcePaths, String outputPath) {
        Map<String, String> opts = new HashMap<>();
        opts.put("caseinsensitivefs", "true");
        opts.put("outputdir", outputPath);
        OptionsImpl options = new OptionsImpl(opts);
        CfrDriver driver = new CfrDriver.Builder().withBuiltOptions(options).build();
        driver.analyse(sourcePaths);
    }

    private static boolean isSpringBootJar(JarFile jar) {
        return jar.getEntry(BOOT_INF) != null;
    }

    private static void extractMavenResources(File jarFile, File outputDir) throws IOException {
        String base = outputDir.getAbsolutePath();
        String resourceDir = base + File.separator + "src" + File.separator + "main" + File.separator + "resources";
        boolean pomExtracted = false;

        JarFile jar = new JarFile(jarFile);
        try {
            boolean isBoot = isSpringBootJar(jar);
            if (isBoot) {
                System.out.println("[JD2M] Detected Spring Boot jar");
                String startClass = readManifestMainClass(jar, "Start-Class");
                if (startClass != null) {
                    System.out.println("[JD2M] Start-Class: " + startClass);
                }
            }

            Enumeration<JarEntry> entries = jar.entries();
            while (entries.hasMoreElements()) {
                JarEntry entry = entries.nextElement();
                String name = entry.getName();
                if (entry.isDirectory()) continue;

                if (!pomExtracted && name.endsWith("pom.xml")) {
                    if (name.matches("META-INF/maven/.*/pom.xml") ||
                            (isBoot && name.matches(BOOT_INF_CLASSES + "META-INF/maven/.*/pom.xml"))) {
                        FileUtil.writeFromStream(jar.getInputStream(entry), base + File.separator + "pom.xml");
                        pomExtracted = true;
                        continue;
                    }
                }

                if (name.endsWith(".class")) continue;

                String resourceName = name;
                if (isBoot) {
                    if (name.startsWith(BOOT_INF_CLASSES)) {
                        resourceName = name.substring(BOOT_INF_CLASSES.length());
                    } else if (name.startsWith(BOOT_INF)) {
                        continue;
                    }
                }
                if (resourceName.isEmpty()) continue;

                boolean skip = false;
                for (String prefix : SKIP_PREFIXES) {
                    if (resourceName.startsWith(prefix)) { skip = true; break; }
                }
                if (skip) continue;

                File dest = new File(resourceDir + File.separator + resourceName);
                FileUtil.writeFromStream(jar.getInputStream(entry), dest);
            }
        } finally {
            jar.close();
        }

        if (!pomExtracted) {
            System.out.println("[JD2M] No pom.xml found, auto-generating...");
            generatePom(jarFile, outputDir);
        }
    }

    private static String readManifestMainClass(JarFile jar, String attrName) {
        try {
            java.util.jar.Manifest mf = jar.getManifest();
            if (mf != null) {
                return mf.getMainAttributes().getValue(attrName);
            }
        } catch (Exception ignored) {
        }
        return null;
    }

    private static void generatePom(File jarFile, File outputDir) throws IOException {
        String groupId = "com.example";
        String artifactId = jarFile.getName();
        int dot = artifactId.lastIndexOf('.');
        if (dot > 0) artifactId = artifactId.substring(0, dot);

        JarFile jar = new JarFile(jarFile);
        try {
            boolean isBoot = isSpringBootJar(jar);
            Map<String, Integer> packageCount = new LinkedHashMap<>();
            Enumeration<JarEntry> entries = jar.entries();
            while (entries.hasMoreElements()) {
                JarEntry entry = entries.nextElement();
                String name = entry.getName();
                if (!name.endsWith(".class")) continue;
                if (isBoot && !name.startsWith(BOOT_INF_CLASSES)) continue;
                int lastSlash = name.lastIndexOf('/');
                if (lastSlash <= 0) continue;
                String pkg = name.substring(0, lastSlash);
                if (isBoot && pkg.startsWith(BOOT_INF_CLASSES)) {
                    pkg = pkg.substring(BOOT_INF_CLASSES.length());
                }
                pkg = pkg.replace('/', '.');
                if (pkg.isEmpty()) continue;
                packageCount.put(pkg, packageCount.getOrDefault(pkg, 0) + 1);
            }
            if (!packageCount.isEmpty()) {
                String bestPkg = packageCount.entrySet().stream()
                        .max(Map.Entry.comparingByValue()).get().getKey();
                int firstDot = bestPkg.indexOf('.');
                groupId = firstDot > 0 ? bestPkg.substring(0, firstDot) : bestPkg;
            }
        } finally {
            jar.close();
        }

        writePomXml(outputDir, groupId, artifactId, "1.0-SNAPSHOT");
    }

    private static void generatePomForClassFile(File classFile, File outputDir) {
        String groupId = "com.example";
        String pkg = getClassPackage(classFile);
        if (!pkg.isEmpty()) {
            int firstDot = pkg.indexOf('.');
            groupId = firstDot > 0 ? pkg.substring(0, firstDot) : pkg;
        }
        String artifactId = classFile.getName().replace(".class", "");
        writePomXml(outputDir, groupId, artifactId, "1.0-SNAPSHOT");
    }

    private static void generatePomForDir(File dir, File outputDir) {
        String groupId = "com.example";
        String artifactId = dir.getName();

        List<String> classFiles = new ArrayList<>();
        collectClassFiles(dir, classFiles);
        if (!classFiles.isEmpty()) {
            String firstClass = classFiles.get(0);
            String pkg = getClassPackage(new File(firstClass));
            if (!pkg.isEmpty()) {
                int firstDot = pkg.indexOf('.');
                groupId = firstDot > 0 ? pkg.substring(0, firstDot) : pkg;
            }
        }
        writePomXml(outputDir, groupId, artifactId, "1.0-SNAPSHOT");
    }

    private static String getClassPackage(File classFile) {
        try {
            byte[] bytes = FileUtil.readBytes(classFile);
            ClassReader cr = new ClassReader(bytes);
            final String[] pkg = {""};
            cr.accept(new ClassVisitor(Opcodes.ASM9) {
                @Override
                public void visit(int version, int access, String name,
                                  String signature, String superName, String[] interfaces) {
                    int lastSlash = name.lastIndexOf('/');
                    if (lastSlash > 0) {
                        pkg[0] = name.substring(0, lastSlash).replace('/', '.');
                    }
                }
            }, ClassReader.SKIP_CODE | ClassReader.SKIP_DEBUG | ClassReader.SKIP_FRAMES);
            return pkg[0];
        } catch (Exception e) {
            return "";
        }
    }

    private static void writePomXml(File outputDir, String groupId, String artifactId, String version) {
        String pom = "<?xml version=\"1.0\" encoding=\"UTF-8\"?>\n" +
                "<project xmlns=\"http://maven.apache.org/POM/4.0.0\"\n" +
                "         xmlns:xsi=\"http://www.w3.org/2001/XMLSchema-instance\"\n" +
                "         xsi:schemaLocation=\"http://maven.apache.org/POM/4.0.0 " +
                "http://maven.apache.org/xsd/maven-4.0.0.xsd\">\n" +
                "    <modelVersion>4.0.0</modelVersion>\n" +
                "\n" +
                "    <groupId>" + groupId + "</groupId>\n" +
                "    <artifactId>" + artifactId + "</artifactId>\n" +
                "    <version>" + version + "</version>\n" +
                "\n" +
                "    <properties>\n" +
                "        <maven.compiler.source>8</maven.compiler.source>\n" +
                "        <maven.compiler.target>8</maven.compiler.target>\n" +
                "        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>\n" +
                "    </properties>\n" +
                "</project>\n";
        FileUtil.writeString(pom, new File(outputDir, "pom.xml"), "UTF-8");
        System.out.println("[JD2M] Generated pom.xml (groupId=" + groupId +
                ", artifactId=" + artifactId + ")");
    }

    private static void convertUnicode(File dir) {
        List<Future<?>> futures = new ArrayList<>();
        collectUnicodeTasks(dir, futures);
        for (Future<?> f : futures) {
            try {
                f.get();
            } catch (Exception ignored) {
            }
        }
        if (!futures.isEmpty()) {
            System.out.println("[JD2M] Unicode -> CN: " + futures.size() + " file(s)");
        }
    }

    private static void collectUnicodeTasks(File dir, List<Future<?>> futures) {
        File[] files = dir.listFiles();
        if (files == null) return;
        for (File f : files) {
            if (f.isDirectory()) {
                collectUnicodeTasks(f, futures);
            } else if (f.getName().endsWith(".java")) {
                futures.add(executor.submit(() -> {
                    try {
                        Unicode2CN.toCN(f);
                    } catch (Exception ignored) {
                    }
                }));
            }
        }
    }

    private static String shortenPath(String path) {
        int idx = Math.max(path.lastIndexOf('/'), path.lastIndexOf('\\'));
        if (idx >= 0 && idx < path.length() - 1) {
            path = path.substring(idx + 1);
        }
        if (path.length() > 80) {
            path = "..." + path.substring(path.length() - 77);
        }
        return path;
    }

    public static class Result {
        public final int success;
        public final int failed;
        public final List<String> failedClasses;

        Result(int success, int failed, List<String> failedClasses) {
            this.success = success;
            this.failed = failed;
            this.failedClasses = failedClasses;
        }
    }
}
```

这个类是整个工具链的**最后一步——反编译**。它把 `.class` 文件反编译成 `.java` 源码，然后把源码中的 `\uXXXX` 转成中文，最终输出一个可导入 IDE 的 Maven 项目。

整体主流程：

```text
1. 入口方法 decompile()
2. 分发方法 decompileFile() / decompileDirectory()
3. 核心反编译 doCfrDecompile()
4. 批量处理 decompileClassList()
5. jar 特殊处理 decompileJarClassByClass()
6. Maven 导出 extractMavenResources() / generatePom()
7. Unicode 转换 convertUnicode()
8. 辅助方法 shortenPath()、Result 类等
```

### 字段

```java
private static final String[] JAR_EXTENSIONS = {".jar", ".war", ".ear"};
private static final String[] SKIP_PREFIXES = {"META-INF", "WEB-INF"};
private static final String BOOT_INF = "BOOT-INF/";
private static final String BOOT_INF_CLASSES = "BOOT-INF/classes/";

private static final ExecutorService executor = new ThreadPoolExecutor(
        8, 8, 60, TimeUnit.SECONDS,
        new LinkedBlockingQueue<>(),
        new ThreadPoolExecutor.CallerRunsPolicy());
```

这几个字段分别用于识别文件类型、过滤不需要的资源、处理 springboot 结构以及提供多线程能力。

#### JAR_EXTENSIONS

| 后缀   | 全称               | 说明                             |
| :----- | :----------------- | :------------------------------- |
| `.jar` | Java Archive       | 标准 Java 库或应用包             |
| `.war` | Web Archive        | Web 应用包（Servlet/JSP 等）     |
| `.ear` | Enterprise Archive | 企业级应用包（包含多个 war/jar） |

#### SKIP_PREFIXES

在提取 jar 包内的非 class 资源文件时，跳过这两个目录。

| 目录        | 内容                    | 为什么跳过                 |
| :---------- | :---------------------- | :------------------------- |
| `META-INF/` | MANIFEST.MF、签名文件等 | 构建信息，不是应用资源     |
| `WEB-INF/`  | web.xml、lib/ 等        | Web 容器配置，不是应用资源 |

跳过这些避免把构建配置和签名文件混入 `src/main/resources`。

#### BOOT_INF 和 BOOT_INF_CLASSES

springboot 可执行 jar 包：

```text
myapp.jar
├── BOOT-INF/
│   ├── classes/
│   │   ├── com/example/UserService.class   ← class 在这里
│   │   └── application.properties          ← 资源在这里
│   └── lib/
│       └── spring-boot-2.7.0.jar           ← 依赖包
├── META-INF/
│   └── MANIFEST.MF
└── org/springframework/boot/loader/        ← Spring Boot 启动器
```

判断是否为 springboot jar

```java
private static boolean isSpringBootJar(JarFile jar) {
    return jar.getEntry(BOOT_INF) != null;
}
```

#### executor - 线程池

```java
private static final ExecutorService executor = new ThreadPoolExecutor(
        8,                              // 核心线程数
        8,                              // 最大线程数
        60, TimeUnit.SECONDS,           // 空闲线程存活时间
        new LinkedBlockingQueue<>(),    // 无界任务队列
        new ThreadPoolExecutor.CallerRunsPolicy()  // 拒绝策略
);
```

| 参数       | 值                            | 含义                                                         |
| :--------- | :---------------------------- | :----------------------------------------------------------- |
| 核心线程数 | 8                             | 保持 8 个线程常驻                                            |
| 最大线程数 | 8                             | 最多 8 个线程（和核心数一样，固定大小）                      |
| 存活时间   | 60 秒                         | 超过核心数的线程空闲 60 秒后回收（这里核心=最大，所以无额外线程） |
| 队列       | `LinkedBlockingQueue`（无界） | 任务排队，不会因为队列满而触发拒绝                           |
| 拒绝策略   | `CallerRunsPolicy`            | 如果线程池已关闭，由提交任务的线程自己执行                   |

### decompile() - 入口

```java
public static Result decompile(File input, File output, boolean mavenExport) throws IOException {
    if (!input.exists()) {  //检查输入是否存在
        throw new IllegalArgumentException("Input not found: " + input.getAbsolutePath());
    }

    long start = System.currentTimeMillis();   //计时开始
    System.out.println("[JD2M] Input : " + input.getAbsolutePath());
    System.out.println("[JD2M] Output: " + output.getAbsolutePath());

    Result result;
    //判断是否是文件或目录，调用对应的方法
    if (input.isFile()) {    
        result = decompileFile(input, output, mavenExport);
    } else {
        result = decompileDirectory(input, output, mavenExport);
    }

    convertUnicode(output);    //先进行反编译，再unicode转中文
    executor.shutdown();   //关闭线程池

    long elapsed = System.currentTimeMillis() - start;
    //打印统计信息
    System.out.println("[JD2M] Done in " + (elapsed / 1000) + "s — " +
            result.success + " OK, " + result.failed + " failed");
	//列出失败清单
    if (!result.failedClasses.isEmpty()) {
        System.out.println("[JD2M] Failed classes (" + result.failedClasses.size() + "):");
        for (String cls : result.failedClasses) {
            System.out.println("  FAIL: " + cls);
        }
    }
    return result;
}
```

| 参数          | 含义                                               |
| :------------ | :------------------------------------------------- |
| `input`       | 输入：.class / .jar / .war / 目录                  |
| `output`      | 输出目录                                           |
| `mavenExport` | 是否生成 Maven 项目结构（pom.xml + src/main/java） |

```java
private static void convertUnicode(File dir) {
    List<Future<?>> futures = new ArrayList<>();
    collectUnicodeTasks(dir, futures);  //收集所有转换任务
    for (Future<?> f : futures) {    //等待所有任务完成
        try {
            f.get();    //阻塞当前线程，待对应任务执行完
        } catch (Exception ignored) {
        }
    }
    if (!futures.isEmpty()) {     //打印统计
        System.out.println("[JD2M] Unicode -> CN: " + futures.size() + " file(s)");
    }
}
```

```java
//递归收集任务
private static void collectUnicodeTasks(File dir, List<Future<?>> futures) {
    File[] files = dir.listFiles();
    if (files == null) return;
    for (File f : files) {
        if (f.isDirectory()) {
            collectUnicodeTasks(f, futures);    //递归到子目录
        } else if (f.getName().endsWith(".java")) {
            futures.add(executor.submit(() -> {    //提交到线程池
                try {
                    Unicode2CN.toCN(f);    //unicode转换
                } catch (Exception ignored) {
                }
            }));
        }
    }
}
```

### decompileFile() - 处理单个文件

```java
private static Result decompileFile(File file, File output, boolean mavenExport) throws IOException {
    String name = file.getName().toLowerCase();
	//单个class文件，直接反编译
    if (name.endsWith(".class")) {
        String javaOutput = output.getAbsolutePath();
        if (mavenExport) {
            javaOutput = output.getAbsolutePath() + File.separator + "src" +
                    File.separator + "main" + File.separator + "java";
            generatePomForClassFile(file, output);
        }
        return decompileSingleClass(file.getAbsolutePath(), javaOutput);
    }
	//是jar/war/ear，解压到临时目录逐个反编译
    for (String ext : JAR_EXTENSIONS) {
        if (name.endsWith(ext)) {
            String javaOutput = output.getAbsolutePath();
            if (mavenExport) {
                javaOutput = output.getAbsolutePath() + File.separator + "src" +
                        File.separator + "main" + File.separator + "java";
            }

            Result result = decompileJarClassByClass(file, new File(javaOutput));
			//导出为maven项目，生成pom.xml
            if (mavenExport) {
                extractMavenResources(file, output);
            }
            return result;
        }
    }

    throw new IllegalArgumentException("Unsupported file type: " + file.getName());
}
```

```java
//单个文件反编译时自动生成pom.xml的方法
private static void generatePomForClassFile(File classFile, File outputDir) {
    String groupId = "com.example";  //默认groupId
    String pkg = getClassPackage(classFile);   //从class文件中读取包名
    if (!pkg.isEmpty()) {
        //提取包名第一段作为groupId
        int firstDot = pkg.indexOf('.');
        groupId = firstDot > 0 ? pkg.substring(0, firstDot) : pkg;  
    }
    String artifactId = classFile.getName().replace(".class", "");
    writePomXml(outputDir, groupId, artifactId, "1.0-SNAPSHOT");//写入pom.xml
}
//用ASM快速读取class文件的包名
private static String getClassPackage(File classFile) {
        try {
            byte[] bytes = FileUtil.readBytes(classFile);
            ClassReader cr = new ClassReader(bytes);
            final String[] pkg = {""};
            cr.accept(new ClassVisitor(Opcodes.ASM9) {
                @Override
                public void visit(int version, int access, String name,
                                  String signature, String superName, String[] interfaces) {
                    int lastSlash = name.lastIndexOf('/');  //获取包名
                    if (lastSlash > 0) {
                        pkg[0] = name.substring(0, lastSlash).replace('/', '.');
                    }
                }
            }, ClassReader.SKIP_CODE | ClassReader.SKIP_DEBUG | ClassReader.SKIP_FRAMES);   //只获取包名
            return pkg[0];  //返回包名
        } catch (Exception e) {
            return "";
        }
    }

private static void writePomXml(File outputDir, String groupId, String artifactId, String version) {
	String pom = "<?xml version=\"1.0\" encoding=\"UTF-8\"?>\n" +
		"<project xmlns=\"http://maven.apache.org/POM/4.0.0\"\n" +
		"         xmlns:xsi=\"http://www.w3.org/2001/XMLSchema-instance\"\n" +
		"         xsi:schemaLocation=\"http://maven.apache.org/POM/4.0.0 " +
		"http://maven.apache.org/xsd/maven-4.0.0.xsd\">\n" +
		"    <modelVersion>4.0.0</modelVersion>\n" +
		"\n" +
		"    <groupId>" + groupId + "</groupId>\n" +
		"    <artifactId>" + artifactId + "</artifactId>\n" +
		"    <version>" + version + "</version>\n" +
		"\n" +
		"    <properties>\n" +
		"        <maven.compiler.source>8</maven.compiler.source>\n" +
		"        <maven.compiler.target>8</maven.compiler.target>\n" +
		"        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>\n" +
		"    </properties>\n" +
		"</project>\n";
	FileUtil.writeString(pom, new File(outputDir, "pom.xml"), "UTF-8");
    //在输出目录下创建pom.xml
	System.out.println("[JD2M] Generated pom.xml (groupId=" + groupId +
	", artifactId=" + artifactId + ")");
}
```

```java
private static Result decompileSingleClass(String classPath, String outputPath) {
    try {
        //方法参数为列表，进行包装
        doCfrDecompile(Collections.singletonList(classPath), outputPath);
        return new Result(1, 0, Collections.emptyList());
    } catch (Exception e) {
        System.err.println("[JD2M] Decompile failed: " + shortenPath(classPath) + " — " + e.getMessage());
        return new Result(0, 1, Collections.singletonList(shortenPath(classPath)));
    }
}
//路径缩短工具，便于日志输出
private static String shortenPath(String path) {
	int idx = Math.max(path.lastIndexOf('/'), path.lastIndexOf('\\'));
	if (idx >= 0 && idx < path.length() - 1) {
		path = path.substring(idx + 1);
	}
	if (path.length() > 80) {
		path = "..." + path.substring(path.length() - 77);
	}
	return path;
}
```

```java
//调用CFR反编译器
private static void doCfrDecompile(List<String> sourcePaths, String outputPath) {
    Map<String, String> opts = new HashMap<>();
    opts.put("caseinsensitivefs", "true");//文件系统大小写不敏感
    opts.put("outputdir", outputPath);//反编译后的java文件输出的目录
    //构建CFR驱动
    OptionsImpl options = new OptionsImpl(opts);
    //CfrDriver.Builder()是构建器模式入口
    CfrDriver driver = new CfrDriver.Builder().withBuiltOptions(options).build();
    //执行反编译
    driver.analyse(sourcePaths);
}
```

```java
//处理jar/war/ear的核心方法
private static Result decompileJarClassByClass(File jarFile, File outputDir) {
    //CFR无法直接读取jar包，先创建临时目录
    Path tempDir = null;
    try {
        tempDir = Files.createTempDirectory("jd2m-");//在系统临时目录下创建
        List<String> classPaths = new ArrayList<>();

        JarFile jar = new JarFile(jarFile);
        try {
            Enumeration<JarEntry> entries = jar.entries();//用来遍历jar包的文件和目录
            while (entries.hasMoreElements()) {
                JarEntry entry = entries.nextElement();
                if (entry.isDirectory() || !entry.getName().endsWith(".class")) continue;
				//把jar中的目录放到临时目录
                File destFile = new File(tempDir.toFile(), entry.getName());
                destFile.getParentFile().mkdirs();//确保父目录存在
                try (InputStream is = jar.getInputStream(entry)) {
                    FileUtil.writeFromStream(is, destFile);
                }
                classPaths.add(destFile.getAbsolutePath());//解压后的绝对路径添加进列表
            }
        } finally {
            jar.close();
        }

        System.out.println("[JD2M] Extracted " + classPaths.size() + " class(es), decompiling...");
		//批量反编译
        Result result = decompileClassList(classPaths, outputDir.getAbsolutePath());
        return result;

    } catch (IOException e) {
        return new Result(0, 0, Collections.singletonList("jar-extract: " + e.getMessage()));
    } finally {
        if (tempDir != null) {
            FileUtil.del(tempDir.toFile());//清理临时目录
        }
    }
}
```

```java
private static Result decompileClassList(List<String> classPaths, String outputPath) {
    int success = 0;
    int failed = 0;
    List<String> failedClasses = new ArrayList<>();
    AtomicInteger progress = new AtomicInteger(0);  //线程安全的进度计数器
    int total = classPaths.size();  //总数

    for (String classPath : classPaths) {
        try {
            doCfrDecompile(Collections.singletonList(classPath), outputPath);
            success++;
        } catch (Exception e) {
            failed++;
            String shortName = shortenPath(classPath);
            failedClasses.add(shortName);
            System.err.println("[JD2M] Decompile failed: " + shortName + " — " + e.getMessage());
        }

        int done = progress.incrementAndGet();
        if (total > 20 && done % (total / 10) == 0) {
            System.out.println("[JD2M] Progress: " + done + "/" + total);
        }
    }
    return new Result(success, failed, failedClasses);
}
```

```java
private static void extractMavenResources(File jarFile, File outputDir) throws IOException {
    String base = outputDir.getAbsolutePath();
    String resourceDir = base + File.separator + "src" + File.separator + "main" + File.separator + "resources";
    boolean pomExtracted = false;  //是否找到pom.xml

    JarFile jar = new JarFile(jarFile);
    try {
        boolean isBoot = isSpringBootJar(jar);//检测springboot结构
        if (isBoot) {
            System.out.println("[JD2M] Detected Spring Boot jar");
            String startClass = readManifestMainClass(jar, "Start-Class");
            if (startClass != null) {
                System.out.println("[JD2M] Start-Class: " + startClass);
            }
        }
		//遍历jar包所有条目
        Enumeration<JarEntry> entries = jar.entries();
        while (entries.hasMoreElements()) {
            JarEntry entry = entries.nextElement();
            String name = entry.getName();
            if (entry.isDirectory()) continue;

            if (!pomExtracted && name.endsWith("pom.xml")) {
                //普通maven构建的jar vs springboot构建的jar
                if (name.matches("META-INF/maven/.*/pom.xml") ||
                        (isBoot && name.matches(BOOT_INF_CLASSES + "META-INF/maven/.*/pom.xml"))) {
                    //找到后直接写入pom.xml
                    FileUtil.writeFromStream(jar.getInputStream(entry), base + File.separator + "pom.xml");
                    pomExtracted = true;
                    continue;
                }
            }

            if (name.endsWith(".class")) continue;
			//处理springboot路径偏移
            String resourceName = name;
            if (isBoot) {
                if (name.startsWith(BOOT_INF_CLASSES)) {
                    resourceName = name.substring(BOOT_INF_CLASSES.length());
                } else if (name.startsWith(BOOT_INF)) {
                    continue;
                }
            }
            if (resourceName.isEmpty()) continue;
			//跳过构建配置目录
            boolean skip = false;
            for (String prefix : SKIP_PREFIXES) {
                if (resourceName.startsWith(prefix)) { skip = true; break; }
            }
            if (skip) continue;
			//写入资源文件
            File dest = new File(resourceDir + File.separator + resourceName);
            FileUtil.writeFromStream(jar.getInputStream(entry), dest);
        }
    } finally {
        jar.close();
    }
	//没有找到pom.xml就自动生成
    if (!pomExtracted) {
        System.out.println("[JD2M] No pom.xml found, auto-generating...");
        generatePom(jarFile, outputDir);
    }
}
```

```java
//从MANIFEST.MF文件中读取指定的属性值，用于获取Spring Boot的启动类名
private static String readManifestMainClass(JarFile jar, String attrName) {
    try {
        java.util.jar.Manifest mf = jar.getManifest();//直接解析META-INF/MANIFEST.MF
        if (mf != null) {
            return mf.getMainAttributes().getValue(attrName);//获取启动类
        }
    } catch (Exception ignored) {
    }
    return null;
}
```

```java
//通过找出所有包名中出现的最多的取第一段作为groupId
private static void generatePom(File jarFile, File outputDir) throws IOException {
    String groupId = "com.example";
    String artifactId = jarFile.getName();
    int dot = artifactId.lastIndexOf('.');
    if (dot > 0) artifactId = artifactId.substring(0, dot);//去掉后缀

    JarFile jar = new JarFile(jarFile);
    try {
        boolean isBoot = isSpringBootJar(jar);
        Map<String, Integer> packageCount = new LinkedHashMap<>();//统计报名出现次数
        Enumeration<JarEntry> entries = jar.entries();
        while (entries.hasMoreElements()) {
            JarEntry entry = entries.nextElement();
            String name = entry.getName();
            if (!name.endsWith(".class")) continue;//只看class文件
            if (isBoot && !name.startsWith(BOOT_INF_CLASSES)) continue;
            int lastSlash = name.lastIndexOf('/');
            if (lastSlash <= 0) continue;
            String pkg = name.substring(0, lastSlash);
            if (isBoot && pkg.startsWith(BOOT_INF_CLASSES)) {
                pkg = pkg.substring(BOOT_INF_CLASSES.length());//提取包结构
            }
            pkg = pkg.replace('/', '.');
            if (pkg.isEmpty()) continue;
            packageCount.put(pkg, packageCount.getOrDefault(pkg, 0) + 1);
        }
        if (!packageCount.isEmpty()) {
            String bestPkg = packageCount.entrySet().stream()
                    .max(Map.Entry.comparingByValue()).get().getKey();
            int firstDot = bestPkg.indexOf('.');
            groupId = firstDot > 0 ? bestPkg.substring(0, firstDot) : bestPkg;
        }
    } finally {
        jar.close();
    }

    writePomXml(outputDir, groupId, artifactId, "1.0-SNAPSHOT");
}
```

### decompileDirectory() - 处理目录

```java
private static Result decompileDirectory(File dir, File output, boolean mavenExport) {
    List<String> classFiles = new ArrayList<>();
    collectClassFiles(dir, classFiles);//递归遍历整个目录树，把所有class文件收集到一个列表中

    if (classFiles.isEmpty()) {
        System.out.println("[JD2M] No .class files found.");
        return new Result(0, 0, Collections.emptyList());
    }

    System.out.println("[JD2M] Found " + classFiles.size() + " .class file(s), decompiling one by one...");
    //输出路径
    String javaOutput = output.getAbsolutePath();  
    if (mavenExport) {
        javaOutput = output.getAbsolutePath() + File.separator + "src" +
                File.separator + "main" + File.separator + "java";
        generatePomForDir(dir, output);//根据目录中的class文件自动推断并生成一个pom.xml
    }
    return decompileClassList(classFiles, javaOutput);//批量反编译
}
```

```java
private static void collectClassFiles(File dir, List<String> result) {
    File[] files = dir.listFiles();
    if (files == null) return;
    for (File f : files) {
        if (f.isDirectory()) {
            collectClassFiles(f, result);//递归调用
        } else if (f.getName().endsWith(".class")) {
            result.add(f.getAbsolutePath());
        }
    }
}
```

```java
private static void generatePomForDir(File dir, File outputDir) {
    String groupId = "com.example";
    String artifactId = dir.getName();

    List<String> classFiles = new ArrayList<>();
    collectClassFiles(dir, classFiles);
    if (!classFiles.isEmpty()) {
        String firstClass = classFiles.get(0);
        String pkg = getClassPackage(new File(firstClass));
        if (!pkg.isEmpty()) {
            int firstDot = pkg.indexOf('.');
            groupId = firstDot > 0 ? pkg.substring(0, firstDot) : pkg;
        }
    }
    writePomXml(outputDir, groupId, artifactId, "1.0-SNAPSHOT");
}
```

```java
private static Result decompileClassList(List<String> classPaths, String outputPath) {
    int success = 0;
    int failed = 0;
    List<String> failedClasses = new ArrayList<>();
    AtomicInteger progress = new AtomicInteger(0);
    int total = classPaths.size();

    for (String classPath : classPaths) {
        try {
            doCfrDecompile(Collections.singletonList(classPath), outputPath);
            success++;
        } catch (Exception e) {
            failed++;
            String shortName = shortenPath(classPath);
            failedClasses.add(shortName);
            System.err.println("[JD2M] Decompile failed: " + shortName + " — " + e.getMessage());
        }

        int done = progress.incrementAndGet();
        if (total > 20 && done % (total / 10) == 0) {
            System.out.println("[JD2M] Progress: " + done + "/" + total);
        }
    }
    return new Result(success, failed, failedClasses);
}
```

这个方法**处理目录输入的反编译**。负责收集目录下所有 `.class` 文件，然后批量反编译。

| 参数          | 含义                             |
| :------------ | :------------------------------- |
| `dir`         | 输入的目录，如 `target/classes/` |
| `output`      | 输出的根目录                     |
| `mavenExport` | 是否生成 Maven 项目结构          |
| 返回值        | 反编译结果（成功/失败数量）      |

## Main.java

```java
package org.jd2m;

import java.io.File;
import java.io.IOException;

public class Main {

    private static final String VERSION = "1.0-SNAPSHOT";

    public static void main(String[] args) {
        if (args.length == 0) {
            printUsage();
            return;
        }

        String command = args[0].toLowerCase();

        switch (command) {
            case "decompile":
            case "d":
                handleDecompile(shiftArgs(args));
                break;
            case "strings":
            case "s":
                handleStrings(shiftArgs(args));
                break;
            case "project":
            case "p":
                handleProject(shiftArgs(args));
                break;
            case "help":
            case "-h":
            case "--help":
                printUsage();
                break;
            case "version":
            case "-v":
            case "--version":
                printVersion();
                break;
            default:
                if (new File(command).exists()) {
                    handleDecompile(args);
                } else {
                    System.err.println("Unknown command: " + command);
                    printUsage();
                }
                break;
        }
    }

    private static String[] shiftArgs(String[] args) {
        String[] result = new String[args.length - 1];
        System.arraycopy(args, 1, result, 0, result.length);
        return result;
    }

    private static void handleDecompile(String[] args) {
        String inputPath = null;
        String outputPath = null;
        boolean mavenExport = false;

        for (int i = 0; i < args.length; i++) {
            switch (args[i]) {
                case "--input":
                case "-i":
                    inputPath = args[++i];
                    break;
                case "--output":
                case "-o":
                    outputPath = args[++i];
                    break;
                case "--maven":
                case "-m":
                    mavenExport = true;
                    break;
                default:
                    if (inputPath == null) {
                        inputPath = args[i];
                    } else if (outputPath == null) {
                        outputPath = args[i];
                    }
                    break;
            }
        }

        if (inputPath == null) {
            System.err.println("Error: No input specified.");
            return;
        }

        File input = new File(inputPath);
        if (outputPath == null) {
            outputPath = deriveOutputName(input);
        }
        File output = new File(outputPath);

        try {
            Decompiler.decompile(input, output, mavenExport);
        } catch (IOException e) {
            System.err.println("[JD2M] Error: " + e.getMessage());
            e.printStackTrace();
        }
    }

    private static void handleStrings(String[] args) {
        String inputPath = null;
        int minLength = 4;

        for (int i = 0; i < args.length; i++) {
            switch (args[i]) {
                case "--min-length":
                case "-n":
                    minLength = Integer.parseInt(args[++i]);
                    if (minLength < 1) {
                        System.err.println("Warning: --min-length must be >= 1, using 4");
                        minLength = 4;
                    }
                    break;
                case "--input":
                case "-i":
                    inputPath = args[++i];
                    break;
                default:
                    if (inputPath == null) {
                        inputPath = args[i];
                    }
                    break;
            }
        }

        if (inputPath == null) {
            System.err.println("Error: No input specified.");
            return;
        }

        StringExtractor.extractFromInput(new File(inputPath), minLength);
    }

    private static void handleProject(String[] args) {
        String jarPath = null;
        String outputPath = null;

        for (int i = 0; i < args.length; i++) {
            switch (args[i]) {
                case "--input":
                case "-i":
                    jarPath = args[++i];
                    break;
                case "--output":
                case "-o":
                    outputPath = args[++i];
                    break;
                default:
                    if (jarPath == null) {
                        jarPath = args[i];
                    } else if (outputPath == null) {
                        outputPath = args[i];
                    }
                    break;
            }
        }

        if (jarPath == null) {
            System.err.println("Error: No input jar specified.");
            return;
        }

        File jarFile = new File(jarPath);
        if (!jarFile.exists() || !jarFile.getName().toLowerCase().endsWith(".jar")) {
            System.err.println("Error: Input must be a .jar file.");
            return;
        }

        if (outputPath == null) {
            outputPath = deriveOutputName(jarFile);
        }
        File output = new File(outputPath);

        try {
            Decompiler.decompile(jarFile, output, true);
            System.out.println("[JD2M] Maven project exported to: " + output.getAbsolutePath());
        } catch (IOException e) {
            System.err.println("[JD2M] Error: " + e.getMessage());
            e.printStackTrace();
        }
    }

    private static String deriveOutputName(File input) {
        String name = input.getName();
        int dot = name.lastIndexOf('.');
        String base = dot > 0 ? name.substring(0, dot) : name;
        return input.getParent() + File.separator + base + "-source";
    }

    private static void printVersion() {
        System.out.println("JD2M version " + VERSION);
    }

    private static void printUsage() {
        System.out.println("JD2M - Java Decompile to Maven & CTF Helper");
        System.out.println();
        System.out.println("Commands:");
        System.out.println("  decompile (d)  Decompile .class/.jar/.war/dir to Java source");
        System.out.println("  strings   (s)  Extract string constants from class files");
        System.out.println("  project   (p)  Export .jar as Maven project (with pom + resources)");
        System.out.println("  help      (h)  Print this help");
        System.out.println("  version   (v)  Print version");
        System.out.println();
        System.out.println("Decompile usage:");
        System.out.println("  jd2m decompile <input> [output] [--maven]");
        System.out.println("    <input>   Jar, .class, .war, or directory");
        System.out.println("    [output]  Output directory (default: <input>-source)");
        System.out.println("    --maven   Export as Maven project (for jars)");
        System.out.println();
        System.out.println("Strings usage:");
        System.out.println("  jd2m strings <input> [--min-length N]");
        System.out.println("    <input>      Jar, .class, or directory");
        System.out.println("    --min-length  Minimum string length (default: 4)");
        System.out.println();
        System.out.println("Examples:");
        System.out.println("  jd2m d target.jar                    # Decompile jar to flat files");
        System.out.println("  jd2m decompile app.jar --maven       # Export as Maven project");
        System.out.println("  jd2m d Foo.class                     # Decompile single class");
        System.out.println("  jd2m d ./classes/                    # Decompile all .class in dir");
        System.out.println("  jd2m s app.jar --min-length 6        # Extract strings (6+ chars)");
        System.out.println("  jd2m s Foo.class                     # Extract strings from a class");
    }
}

```

### main() - 入口

```java
public static void main(String[] args) {
    if (args.length == 0) {
        printUsage();
        return;
    }
    String command = args[0].toLowerCase();
    switch (command) {
        case "decompile": case "d": → handleDecompile()
        case "strings":   case "s": → handleStrings()
        case "project":   case "p": → handleProject()
        case "help":                 → printUsage()
        case "version":              → printVersion()
        default:                     → 如果第一个参数是存在的文件路径，自动识别为反编译
    }
}
```

**三种核心命令**：

| 命令        | 简写 | 功能                    | 调用的类                              |
| :---------- | :--- | :---------------------- | :------------------------------------ |
| `decompile` | `d`  | 反编译为 Java 源码      | `Decompiler`                          |
| `strings`   | `s`  | 提取字节码中的字符串    | `StringExtractor`                     |
| `project`   | `p`  | 反编译并导出 Maven 项目 | `Decompiler`（强制 mavenExport=true） |

### handleDecompile()

```java
private static void handleDecompile(String[] args) {
    String inputPath = null;
    String outputPath = null;
    boolean mavenExport = false;

    for (int i = 0; i < args.length; i++) {
        switch (args[i]) {
            case "--input":
            case "-i":
                inputPath = args[++i];
                break;
            case "--output":
            case "-o":
                outputPath = args[++i];
                break;
            case "--maven":
            case "-m":
                mavenExport = true;
                break;
            default:
                if (inputPath == null) {
                    inputPath = args[i];
                } else if (outputPath == null) {
                    outputPath = args[i];
                }
                break;
        }
    }

    if (inputPath == null) {
        System.err.println("Error: No input specified.");
        return;
    }

    File input = new File(inputPath);
    if (outputPath == null) {
        outputPath = deriveOutputName(input);
    }
    File output = new File(outputPath);

    try {
        Decompiler.decompile(input, output, mavenExport);
    } catch (IOException e) {
        System.err.println("[JD2M] Error: " + e.getMessage());
        e.printStackTrace();
    }
}
```

**支持的参数**：

| 参数       | 简写 | 作用                           |
| :--------- | :--- | :----------------------------- |
| `--input`  | `-i` | 输入文件/目录                  |
| `--output` | `-o` | 输出目录（可选，默认自动生成） |
| `--maven`  | `-m` | 是否导出为 Maven 项目          |

### handleStrings()

```java
private static void handleStrings(String[] args) {
    String inputPath = null;
    int minLength = 4;

    for (int i = 0; i < args.length; i++) {
        switch (args[i]) {
            case "--min-length":
            case "-n":
                minLength = Integer.parseInt(args[++i]);
                if (minLength < 1) {
                    System.err.println("Warning: --min-length must be >= 1, using 4");
                    minLength = 4;
                }
                break;
            case "--input":
            case "-i":
                inputPath = args[++i];
                break;
            default:
                if (inputPath == null) {
                    inputPath = args[i];
                }
                break;
        }
    }

    if (inputPath == null) {
        System.err.println("Error: No input specified.");
        return;
    }

    StringExtractor.extractFromInput(new File(inputPath), minLength);
}
```

**支持的参数**：

| 参数           | 简写 | 作用                     |
| :------------- | :--- | :----------------------- |
| `--input`      | `-i` | 输入文件/目录            |
| `--min-length` | `-n` | 最小字符串长度（默认 4） |

### handleProject()

```java
private static void handleProject(String[] args) {
    String jarPath = null;
    String outputPath = null;

    for (int i = 0; i < args.length; i++) {
        switch (args[i]) {
            case "--input":
            case "-i":
                jarPath = args[++i];
                break;
            case "--output":
            case "-o":
                outputPath = args[++i];
                break;
            default:
                if (jarPath == null) {
                    jarPath = args[i];
                } else if (outputPath == null) {
                    outputPath = args[i];
                }
                break;
        }
    }

    if (jarPath == null) {
        System.err.println("Error: No input jar specified.");
        return;
    }

    File jarFile = new File(jarPath);
    if (!jarFile.exists() || !jarFile.getName().toLowerCase().endsWith(".jar")) {
        System.err.println("Error: Input must be a .jar file.");
        return;
    }

    if (outputPath == null) {
        outputPath = deriveOutputName(jarFile);
    }
    File output = new File(outputPath);

    try {
        Decompiler.decompile(jarFile, output, true);
        System.out.println("[JD2M] Maven project exported to: " + output.getAbsolutePath());
    } catch (IOException e) {
        System.err.println("[JD2M] Error: " + e.getMessage());
        e.printStackTrace();
    }
}
```

- 强制要求输入是 `.jar` 文件
- 强制开启 Maven 导出模式（`mavenExport=true`）

### shiftArgs()

```java
// 去掉第一个元素（命令名），返回剩余参数
private static String[] shiftArgs(String[] args) {
    String[] result = new String[args.length - 1];
    System.arraycopy(args, 1, result, 0, result.length);
    return result;
}
```

### deriveOutputName()

```java
private static String deriveOutputName(File input) {
    String name = input.getName();
    int dot = name.lastIndexOf('.');
    String base = dot > 0 ? name.substring(0, dot) : name;
    return input.getParent() + File.separator + base + "-source";
}
```

自动生成输出目录名：去掉文件后缀，加 `-source`。

核心功能的调用链：

```text
main()
  │
  ├── "decompile" → handleDecompile()
  │     └── Decompiler.decompile(input, output, mavenExport)
  │           ├── .class → decompileSingleClass → doCfrDecompile → CFR
  │           ├── .jar   → decompileJarClassByClass → decompileClassList → CFR
  │           └── 目录   → collectClassFiles → decompileClassList → CFR
  │           └── convertUnicode → Unicode2CN.toCN
  │
  ├── "strings" → handleStrings()
  │     └── StringExtractor.extractFromInput(input, minLength)
  │           ├── .class → extractFromClass → ASM 收集
  │           ├── .jar   → extractFromJar → 遍历 + ASM 收集
  │           └── 目录   → extractFromDir → 递归 + ASM 收集
  │
  └── "project" → handleProject()
        └── Decompiler.decompile(jarFile, output, true)  ← 同上，强制 mavenExport
```



































