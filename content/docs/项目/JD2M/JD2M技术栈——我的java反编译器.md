---
title:  JD2M技术栈——我的Java反编译器
date: 2026-07-03
---

# JD2M技术栈——我的 Java 反编译器

## 前置知识：Java 从源码到运行

```text
.java 源码
    │
    ▼ javac 编译
.class 字节码文件（平台无关的二进制格式）
    │
    ▼ java 命令启动 JVM
JVM 加载 → 验证 → 解释执行 / JIT 编译 → 机器码
```

**反编译就是把流程反过来：**

```text
.class 字节码
    │
    ▼ 反编译器（如 CFR）
.java 源码（近似还原，不可能 100% 还原）
```

### 为什么反编译不能百分百还原

编译器在编译过程中会丢失一些信息：

| 丢失的信息         | 举例                                         |
| ------------------ | -------------------------------------------- |
| 局部变量名         | `int a` ← 字节码中无变量名，只有类型和栈位置 |
| 泛型参数           | `List<String>` → `List`（泛型擦除）          |
| 注释               | 所有注释在编译时被丢弃                       |
| 格式/缩进          | 完全不存在                                   |
| for/while 区别     | 都变成相同条件的 goto 跳转                   |
| switch 字符串      | 编译为 `hashCode()` 比较 + 跳转表            |
| try-with-resources | 编译为 try-catch-finally 模式                |

## class 文件格式

每一个`.class`文件都严格按照一下结构排列：

```text
ClassFile {
    u4             magic;              // 0xCAFEBABE（魔数，标识这是一个 class 文件）
    u2             minor_version;      // 次版本号
    u2             major_version;      // 主版本号（52 = Java 8, 55 = Java 11）
    u2             constant_pool_count;
    cp_info        constant_pool[constant_pool_count-1];  // 常量池
    u2             access_flags;       // public/abstract/final 等修饰符
    u2             this_class;         // 指向常量池，本类名
    u2             super_class;        // 指向常量池，父类名
    u2             interfaces_count;
    u2             interfaces[interfaces_count];  // 实现的接口
    u2             fields_count;
    field_info     fields[fields_count];          // 字段表
    u2             methods_count;
    method_info    methods[methods_count];        // 方法表
    u2             attributes_count;
    attribute_info attributes[attributes_count];  // 属性表（包括 Code 属性）
}
```

在 JVM 规范中，`u4`、`u2`、`u1` 是描述 Class 文件格式时使用的**基本数据类型**，用来定义每个字段占用的字节数。而 cp_info、field_info、method_info、attribute_info 这四个都是 Class 文件中定义复杂结构的**伪结构（或叫数据类型）**。

它们不像 `u2` 那样只代表一个数值，而是**一个由多个基本类型组合成的复合结构**，用于承载常量、字段、方法和代码等核心信息。

- `cp_info` (常量项)
  常量池中的每一项。它是个“标签联合体”，结构取决于第一个字节的 `tag`。比如 `tag` 为 1 代表文本字符串，后面存长度和内容；`tag` 为 7 代表类符号引用，后面存指向全限定名的索引。

  ```text
  cp_info {
      u1 tag;   // <-- 就是这个
      u1 info[]; // 后面是具体内容，长度和结构由 tag 决定
  }
  ```

  简单说，**`tag` 就是常量池每一项的第一个字节，是一个用于“区分类型”的标志数字。**它本身就是一个 `u1` 类型（1 字节无符号整数），用不同的数值代表不同的常量类型。JVM 规范定义了这些数字对应的类型：

  | tag 值 | 常量类型                      | 说明                 |
  | :----- | :---------------------------- | :------------------- |
  | **1**  | `CONSTANT_Utf8`               | UTF-8 编码的字符串   |
  | **3**  | `CONSTANT_Integer`            | 4 字节整型常量       |
  | **4**  | `CONSTANT_Float`              | 4 字节浮点常量       |
  | **5**  | `CONSTANT_Long`               | 8 字节长整型常量     |
  | **6**  | `CONSTANT_Double`             | 8 字节双精度浮点常量 |
  | **7**  | `CONSTANT_Class`              | 类或接口的符号引用   |
  | **8**  | `CONSTANT_String`             | 字符串类型字面量     |
  | **9**  | `CONSTANT_Fieldref`           | 字段的符号引用       |
  | **10** | `CONSTANT_Methodref`          | 方法的符号引用       |
  | **11** | `CONSTANT_InterfaceMethodref` | 接口方法的符号引用   |
  | **12** | `CONSTANT_NameAndType`        | 名称和类型描述符     |
  | **15** | `CONSTANT_MethodHandle`       | 方法句柄             |
  | **16** | `CONSTANT_MethodType`         | 方法类型             |
  | **17** | `CONSTANT_Dynamic`            | 动态计算常量         |
  | **18** | `CONSTANT_InvokeDynamic`      | 动态调用点           |
  | **19** | `CONSTANT_Module`             | 模块信息             |
  | **20** | `CONSTANT_Package`            | 包信息               |
  
- `field_info` (字段表)
  字段表，就是 `field_info` 结构组成的数组，用来描述**类或接口中声明的所有字段**，包括静态变量、实例变量、以及接口中的常量。
  一个字段表的结构如下：
  
  ```text
  field_info {
      u2             access_flags;       // 访问标志
      u2             name_index;         // 字段简单名称，指向常量池
      u2             descriptor_index;   // 字段描述符，指向常量池
      u2             attributes_count;   // 属性数量
      attribute_info attributes[attributes_count]; // 属性表
  }
  ```
  
- `method_info` (方法表)
  描述类或接口中声明的**方法**。结构和 `field_info` 很像，包括访问标志、方法名索引、描述符索引（如 `()V` 代表无参无返回）。它最重要的属性是 **`Code` 属性**，里面就存放着方法体编译后的字节码指令。

- `attribute_info` (属性表)
  一种可扩展的通用结构，附属在上面几种结构里，用于存放额外信息。比如方法里的字节码存在 `Code` 属性，方法签名异常声明在 `Exceptions` 属性，源码行号调试信息在 `LineNumberTable` 属性等。它也是由名字索引、长度和具体内容组成的。

### 常量池

常量池是 class 文件的最核心部分，其中存放了两大类信息：

**字面量：**

- 字符串常量
- 数值常量
- `null`

字面量其实是字段的值，比如`int a=5`，这里的`5`就是字面量

**符号引用：**

- 类和接口的全限定名，比如`com/example/demo`
- 字段名称和描述符`name:I`（字段 name，类型 int）
- 方法名称和描述符`toString:()Ljava/lang/String;`
- 方法句柄、方法类型

方法描述符的格式是 `(参数字段描述符)返回值字段描述符`，也就是参数和返回值都用你之前学过的字段描述符来表示：

- **`()`**：括号里的内容是参数列表。这里为空，表示这个方法没有参数。
- **`Ljava/lang/String;`**：紧跟在括号后面的，是返回值类型。这里表示返回一个 `String` 对象。

举个有参数的方法：`int indexOf(String str, int fromIndex)`。

它的简写就是 `indexOf:(Ljava/lang/String;I)I`

方法类型描述一个方法长什么样，有什么参数，返回什么类型；方法句柄则可以直接指向并调用一个具体的方法。

**常量池条目类型（部分）：**

| 标签 | 类型                 | 存储内容                             |
| ---- | -------------------- | ------------------------------------ |
| 1    | CONSTANT_Utf8        | UTF-8 编码的字符串                   |
| 3    | CONSTANT_Integer     | 4 字节整数                           |
| 7    | CONSTANT_Class       | 类的全限定名（指向 Utf8）            |
| 9    | CONSTANT_Fieldref    | 字段引用（指向 Class + NameAndType） |
| 10   | CONSTANT_Methodref   | 方法引用                             |
| 12   | CONSTANT_NameAndType | 方法/字段名 + 描述符                 |

### 描述符

JVM 使用紧凑的字符串表示类型：

| Java 类型  | 描述符                |
| ---------- | --------------------- |
| `int`      | `I`                   |
| `long`     | `J`                   |
| `float`    | `F`                   |
| `double`   | `D`                   |
| `boolean`  | `Z`                   |
| `char`     | `C`                   |
| `void`     | `V`                   |
| `Object`   | `Ljava/lang/Object;`  |
| `String[]` | `[Ljava/lang/String;` |
| `int[][]`  | `[[I`                 |

方法描述符：`(参数描述符)返回值描述符`

```
int add(int a, long b)   →  (IJ)I
void main(String[] args) →  ([Ljava/lang/String;)V
```

## JVM 字节码指令集

JVM 是基于栈的虚拟机，所有操作在操作数栈上进行。这里的栈都是指操作数栈。JVM 执行指令时，并不是直接操作 CPU 寄存器或内存，而是通过一个叫“操作数栈”的中间区域来传递数据和进行计算。

```text
Java 源码:          int c = a + b;

编译为字节码:
    iload_1         // 将局部变量 1 (a) 压入操作数栈顶
    iload_2         // 将局部变量 2 (b) 压入操作数栈顶
    iadd            // 弹出栈顶两个 int，相加，结果压回栈
    istore_3        // 将栈顶结果存入局部变量 3 (c)
```

上面的例子就是一个简单的 JVM 指令。

**指令分类：**

### 加载/存储指令

```text
aload       从局部变量加载引用到栈
iload       加载 int
lload       加载 long
fload       加载 float
dload       加载 double
astore      将栈顶引用存到局部变量
bipush      将 byte 扩展为 int 压栈
iconst_0    将常量 0 压栈（-1 到 5 都有独立指令）
ldc         从常量池加载常量（int/float/String/Class）
```

### 算术指令

```text
iadd        两个 int 相加
isub        相减
imul        相乘
idiv        相除
irem        取余
iinc        局部变量自增 (i++)
```

### 类型转换

```text
i2l         int → long
i2f         int → float
l2i         long → int
d2i         double → int
checkcast   类型检查转换
```

### 对象操作

```text
new         创建对象（分配内存，不调用构造器）
newarray    创建基本类型数组
anewarray   创建引用类型数组
getfield    获取实例字段
putfield    设置实例字段
getstatic   获取静态字段
putstatic   设置静态字段
```

### 方法调用

```text
invokevirtual    调用实例方法（虚方法分发）
invokespecial    调用构造器/私有方法/父类方法
invokestatic     调用静态方法
invokeinterface  调用接口方法
invokedynamic    动态方法调用（lambda、字符串拼接）
```

### 控制流

```text
goto            无条件跳转
ifeq            等于 0 时跳转
ifne            不等于 0 时跳转
if_icmpeq       两个 int 比较，相等跳转
if_icmpne       不相等跳转
tableswitch     表跳转（连续 case 值）
lookupswitch    查找跳转（稀疏 case 值）
```

### 返回指令

```text
return          返回 void
ireturn         返回 int
areturn         返回引用
```

### 异常

```text
athrow          抛出异常
```

### 同步

```text
monitorenter    进入 synchronized 块
monitorexit     退出 synchronized 块
```

**完整字节码示例：**

```java
// 源码
public static String checkFlag(String input) {
    if (input.equals("FLAG{secret}")) {
        return "Correct!";
    }
    return "Wrong!";
}
```

```text
// 对应的字节码（javap -c 输出）
public static java.lang.String checkFlag(java.lang.String);
  Code:
     0: aload_0                     // 加载参数 input
     1: ldc #2                      // 从常量池加载 "FLAG{secret}"
     3: invokevirtual #3            // 调用 String.equals()
     6: ifeq 14                     // 结果为 false 跳转到 14
     9: ldc #4                      // 加载 "Correct!"
    11: areturn                     // 返回 "Correct!"
    14: ldc #5                      // 加载 "Wrong!"
    16: areturn                     // 返回 "Wrong!"
```

## CFR 反编译器原理

CFR 是目前公认的最准确的 Java 反编译器之一。核心思想：

### 阶段1：字节码读取

将 class 文件的字节码转换为内部数据结构。CFR 读取完字节码之后转化成 CFR 能够处理的一个个类和对象。

### 阶段2：控制流程（CFG）重建

```text
字节码是扁平的指令序列（带跳转标签）
       │
       ▼
CFR 分析跳转指令（goto/ifeq/tableswitch 等）
       │
       ▼
构建基本块（连续执行的一段指令）
       │
       ▼
连接基本块形成控制流图（CFG）
```

**示例：if-else 的 CFG**

```text
    [Block 0: 条件判断]
           │
      ┌────┴────┐
      ▼         ▼
[Block 1: if-true]   [Block 2: if-false]
      │               │
      └──────┬────────┘
             ▼
    [Block 3: 汇合点]
```

### 阶段 3：结构化还原

这是 CFR 最核心的技术。字节码中只有 goto 跳转，CFR 通过模式识别还原高级结构：

| 字节码模式                  | 还原的 Java 结构    |
| --------------------------- | ------------------- |
| 条件跳转 + 循环回到顶部     | `while(condition)`  |
| 条件跳转 + 循环到底部       | `do-while`          |
| 初始化 + 条件 + 递增 + 循环 | `for`               |
| 条件链跳转                  | `if-else if-else`   |
| tableswitch/lookupswitch    | `switch-case`       |
| try 块 + 异常表             | `try-catch-finally` |

### 阶段 4：表达式重建

JVM 的栈式操作需要还原为表达式树：

```text
字节码:
    iload_1     // a
    iload_2     // b
    iadd        // a + b
    iload_3     // c
    imul        // (a + b) * c
    iconst_1    // 1
    iadd        // (a + b) * c + 1

还原为:
    (a + b) * c + 1
```

CFR 通过模拟操作数栈，将一连串的压栈/弹栈操作合并为表达式。

### 阶段 5：Lambda 和内建方法还原

Java 8 引入的 lambda 编译后会生成 `invokedynamic` 指令和合成方法。CFR 识别这些模式并还原为原始 lambda 表达式。

### 阶段6：代码美化输出

将内部数据结构格式化为可读的 Java 源文件。

## ASM 字节码框架

ASM 是一个 JVM 字节码操作和分析框架，采用 **访问者模式（Visitor Pattern）** 来解析和生成 class 文件。

### 核心架构

```
ClassReader（读取字节码）
    │
    ▼ accept()
ClassVisitor（访问类结构）
    │
    ├── visit()             类头部信息
    ├── visitField()       字段 → FieldVisitor
    ├── visitMethod()      方法 → MethodVisitor
    │       │
    │       ├── visitCode()              方法开始
    │       ├── visitLdcInsn()           LDC 指令（加载常量）
    │       ├── visitMethodInsn()        方法调用指令
    │       ├── visitJumpInsn()          跳转指令
    │       ├── visitVarInsn()           局部变量操作
    │       ├── visitMaxs()              栈和局部变量最大数量
    │       └── visitEnd()              方法结束
    └── visitEnd()          类结束
```

ASM 逐个字节解析 class 文件，每遇到一个结构就调用 visitor 的对应方法。这种**事件驱动**的方式非常高效——不需要把整个 class 文件加载成对象树，解析完就调用回调，内存开销极低。

### 为什么用 ASM 而不是其他字节码框架

| 框架       | 特点                             | 适用场景                |
| ---------- | -------------------------------- | ----------------------- |
| **ASM**    | 轻量（~120KB）、高性能、事件驱动 | 需要速度的分析/生成工具 |
| BCEL       | 完整对象模型、重                 | 需要修改字节码的工具    |
| Javassist  | 源码级 API、简单                 | 运行时代码生成          |
| Byte Buddy | 高级 DSL、功能全面               | 现代字节码生成和代理    |

JD2M 选择 ASM 的原因：对 class 文件做只读分析（不需要修改、生成），ASM 的 visitor 模式零额外开销，并且只引入了 120KB 的依赖体积。



























