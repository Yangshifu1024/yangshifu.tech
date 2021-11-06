---
title: "Java 字节码结构"
date: 2019-04-03T21:47:45+08:00
draft: false
categories:
  - JVM
tags:
  - Java
  - JVM
  - bytecode
---

<!-- more -->

## 字节码结构



|     区域     | 大小（字节） |           说明           |
| :----------: | :----------: | :----------------------: |
|     魔数     |      4       |    固定为 0xCAFEBABE     |
|   副版本号   |      2       |      Minor Version       |
|   主版本号   |      2       |      Majar Version       |
| 常量池计数器 |      2       |        常量池大小        |
| 常量池数据区 |              |     字面量和符号引用     |
|   访问标志   |      2       |       表示访问权限       |
|    类索引    |      2       |   确定当前类的全限定名   |
|   超类索引   |      2       | 确定当前类的超类全限定名 |
|  接口计数器  |      2       |         接口数量         |
|    接口表    |              |     接口索引数组集合     |
|  字段计数器  |      2       |         字段数量         |
|    字段表    |              |         字段数组         |
|  方法计数器  |      2       |     当前类的方法数量     |
|    方法表    |              |       方法数组集合       |
|  属性计数器  |      2       |         属性数量         |
|    属性表    |              |       属性数组集合       |

<!--more-->

## 魔数
一个有效的 Java 字节码文件的前 4 个字节固定为：`0XCAFEBABE`，其他同样使用魔数的文件类型为：`jpeg`、`png`、`gif` 等。

JVM 用魔数来校验文件是否是一个有效且合法的字节码文件。

## 版本号
主版本号（M）和次版本号（m）限定了字节码文件支持的 JVM 版本。

## 常量池计数器和常量池

### 概述

常量池中的有效数据是从下标 1 开始的，0 下标保留，则常量池计数器也是从 1 开始的。

常量池中主要用于存放字面量和符号引用，其中：

字面量由文字字符串、final 常量值等构成；
符号引用则包括了类和接口的全限定名（Fully Qualified Name，FQN）、字段的名称和描述符（Descriptor），以及方法的名称和描述符。

假设有如下代码：

```java
// Test.java
package com.example;

public class Test {
  public final double j = 1.23;
  public int i = 0;
  private String s = "HelloWorld!";
  public void hello() {
    System.out.println(s);
  }
}
```

首先将其编译为 Java 字节码：`javac Test.java`，然后使用 javap 工具查看其字节码内容：`javap -v Test`，截取 `Constant pool` 段如下：

```java
   #1 = Methodref          #11.#26        // java/lang/Object."<init>":()V
   #2 = Double             1.23d
   #4 = Fieldref           #10.#27        // com/example/Test.j:D
   #5 = Fieldref           #10.#28        // com/example/Test.i:I
   #6 = String             #29            // HelloWorld!
   #7 = Fieldref           #10.#30        // com/example/Test.s:Ljava/lang/String;
   #8 = Fieldref           #31.#32        // java/lang/System.out:Ljava/io/PrintStream;
   #9 = Methodref          #33.#34        // java/io/PrintStream.println:(Ljava/lang/String;)V
  #10 = Class              #35            // com/example/Test
  #11 = Class              #36            // java/lang/Object
  #12 = Utf8               j
  #13 = Utf8               D
  #14 = Utf8               ConstantValue
  #15 = Utf8               i
  #16 = Utf8               I
  #17 = Utf8               s
  #18 = Utf8               Ljava/lang/String;
  #19 = Utf8               <init>
  #20 = Utf8               ()V
  #21 = Utf8               Code
  #22 = Utf8               LineNumberTable
  #23 = Utf8               hello
  #24 = Utf8               SourceFile
  #25 = Utf8               Test.java
  #26 = NameAndType        #19:#20        // "<init>":()V
  #27 = NameAndType        #12:#13        // j:D
  #28 = NameAndType        #15:#16        // i:I
  #29 = Utf8               HelloWorld!
  #30 = NameAndType        #17:#18        // s:Ljava/lang/String;
  #31 = Class              #37            // java/lang/System
  #32 = NameAndType        #38:#39        // out:Ljava/io/PrintStream;
  #33 = Class              #40            // java/io/PrintStream
  #34 = NameAndType        #41:#42        // println:(Ljava/lang/String;)V
  #35 = Utf8               com/example/Test
  #36 = Utf8               java/lang/Object
  #37 = Utf8               java/lang/System
  #38 = Utf8               out
  #39 = Utf8               Ljava/io/PrintStream;
  #40 = Utf8               java/io/PrintStream
  #41 = Utf8               println
  #42 = Utf8               (Ljava/lang/String;)V
```

第 #1 行表示存储的是一个方法引用，指向了 #11 和 #26，并用`.`连接它们；

找到 #11，它是一个类的符号引用，指向了 #36；找到 #26 ，它是一个字段或方法的符号引用，指向了 #19 和 #20，用`:`连接；

找到 #36，它是一个 UTF-8 编码的字符串：`java/lang/Object`；找到 #19 和 #20，它们都是 UTF-8 编码的字符串`<init>:()V`；

最终得到 #1 行的方法引用指向了`java/lang/Object."<init>":()V`，其他以此类推。

### 常量池中的表类型：

|     javap 结果     |           类型           |
| :----------------: | :----------------------: |
|        Utf8        |    UTF-8 编码的字符串    |
|      Integer       |        整形字面值        |
|       Float        |    单精度浮点数字面值    |
|        Long        |       长整型字面值       |
|       Double       |    双精度副店数字面值    |
|       Class        |    类或接口的符号引用    |
|       String       |    字符串类型的字面值    |
|      Fieldref      |      字段的符号引用      |
|     Methodref      |      方法的符号引用      |
| InterfaceMethodref |   接口中方法的符号引用   |
|    NameAndType     | 字段或方法的部分符号引用 |
|    MethodHandle    |         方法句柄         |
|     MethodType     |         方法类型         |
|   InvokeDynamic    |      动态方法调用点      |

## 访问标志

常量池之后的两个字节存储的就是访问标志（access flags），主要用于表示该类或接口的访问权限。

有如下代码：
```java
// Public.java
package com.example;
public class Public {}
```
编译为字节码后用 `javap -v Public` 查看，截取访问标志段如下：
```java
flags: ACC_PUBLIC, ACC_SUPER
```
代表该类声明为了 `public`，可被包外的类访问。

|    标志名称    |   值   |                        描述                         |
| :------------: | :----: | :-------------------------------------------------: |
|   ACC_PUBLIC   | 0x0001 |             声明为 public，可从包外访问             |
|   ACC_FINAL    | 0x0010 |              声明为 final，不允许派生               |
|   ACC_SUPER    | 0x0020 | 当用到 invokespecial 指令时，需要特殊处理的超类方法 |
| ACC_INTERFACE  | 0x0200 |              标志定义的是接口而不是类               |
|  ACC_ABSTRACT  | 0x0400 |            声明为 abstract，不能被实例化            |
| ACC_SYNTHETIC  | 0x1000 |     声明为 synthetic，标识并非由 Java 代码生成      |
| ACC_ANNOTATION | 0x2000 |                    标识注解类型                     |
|    ACC_ENUM    | 0x4000 |                    标识枚举类型                     |

## 类索引

当前类索引和超类索引都是指向常量池中的 Class 类型，Class 类型又指向 Utf8 类型，最终确定类的全限定名。

Java 语法规范规定只允许继承一个类，如果当前类没有显式继承任何超类时，则它的直接超类就是 Object，父类索引为 0，即指向了常量池中的下标 0 保留项。

## 接口计数器和接口表

接口计数器用于表示当前类或者接口的直接超类接口数量；接口表实际是一个数组集合，它包含了当前类或接口的直接超类在常量池中的索引集合，通过这个索引即可确定当前类或接口的超类接口全限定名。

## 字段计数器和字段表

字段计数器标识一个字节码文件中的字段总数，包括类变量和实例变量；字段表则是一个数组集合，其中的每个成员都表示了一个字段的完整信息。

有如下代码：
```java
// Types.java
public class Types {

  public static String s = "HelloWorld!";
  public final double pi = 3.14;
  public int i = 123;
  protect Integer j = 456;
  private Integer[] arr = new Integer[]{7, 8, 9};
}
```
编译为字节码后通过 `javap` 查看：`javap -p -v Types`，截取字段部分如下：
```java
  public static java.lang.String s;
    descriptor: Ljava/lang/String;
    flags: ACC_PUBLIC, ACC_STATIC

  public final double pi;
    descriptor: D
    flags: ACC_PUBLIC, ACC_FINAL
    ConstantValue: double 3.14d

  public int i;
    descriptor: I
    flags: ACC_PUBLIC

  protected java.lang.Integer j;
    descriptor: Ljava/lang/Integer;
    flags: ACC_PROTECTED

  private java.lang.Integer[] arr;
    descriptor: [Ljava/lang/Integer;
    flags: ACC_PRIVATE
```
其中每项都完整描述了该字段的相关信息，如描述符（descriptor）、访问标志（flags），甚至被声明为 final 的字段还携带了常量值（ConstantValue）。

字段描述符总结如下：

| Descriptor |   类型    |    含义    |
| :--------: | :-------: | :--------: |
|     B      |   byte    |    字节    |
|     C      |   char    |    字符    |
|     D      |  double   | 双精度浮点 |
|     F      |   float   | 单精度浮点 |
|     I      |    int    |    整形    |
|     J      |   long    |   长整型   |
|     S      |   short   |   短整型   |
|     Z      |  boolean  |    布尔    |
|     L      | reference |  对象类型  |
|     [      | reference |  数组类型  |

有如上总结表后，可得知：

上例中的 j，描述符为 `Ljava/lang/Integer`，即代表 `Integer` 对象；arr 描述符为 `[Ljava/lang/Integer`，即代表 `Integer` 数组。

## 方法计数器和方法表

方法计数器代表该类中的方法总数，仅包含该类或接口定义的方法，不包含继承而来的方法。

方法表则是一个数组集合，其中每项都表示了一个方法的完整信息。

有如下代码：
``` java
// Methods.java
public abstract class Methods {
  public int intPublic() {
    return 1;
  }

  private String strPrivate() {
    return "HelloWorld!";
  }

  protected Object objectProtected() {
    return new Object();
  }

  public static void voidStatic() {

  }

  public final void finalPublic() {

  }

  public synchronized void syncPublic() {

  }

  private void dynamicArgs(String first, Object... args) {

  }

  public abstract void abstractPublic();
}
```

编译为字节码后使用 javap 查看，并截取方法定义部分（仅保留描述符和访问标志）如下：
``` java
  public Methods();
    descriptor: ()V
    flags: ACC_PUBLIC

  public int intPublic();
    descriptor: ()I
    flags: ACC_PUBLIC

  private java.lang.String strPrivate();
    descriptor: ()Ljava/lang/String;
    flags: ACC_PRIVATE

  protected java.lang.Object objectProtected();
    descriptor: ()Ljava/lang/Object;
    flags: ACC_PROTECTED

  public static void voidStatic();
    descriptor: ()V
    flags: ACC_PUBLIC, ACC_STATIC

  public final void finalPublic();
    descriptor: ()V
    flags: ACC_PUBLIC, ACC_FINAL

  public synchronized void syncPublic();
    descriptor: ()V
    flags: ACC_PUBLIC, ACC_SYNCHRONIZED

  private void dynamicArgs(java.lang.String, java.lang.Object...);
    descriptor: (Ljava/lang/String;[Ljava/lang/Object;)V
    flags: ACC_PRIVATE, ACC_VARARGS

  public abstract void abstractPublic();
    descriptor: ()V
    flags: ACC_PUBLIC, ACC_ABSTRACT
```

方法访问标志信息如下表：

|     标志名称     |          描述           |
| :--------------: | :---------------------: |
|    ACC_PUBLIC    |    声明方法为 public    |
|   ACC_PRIVATE    |   声明方法为 private    |
|  ACC_PROTECTED   |  声明方法为 protected   |
|    ACC_STATIC    |    声明方法为类方法     |
|    ACC_FINAL     |    声明方法为 final     |
| ACC_SYNCHRONIZED | 声明方法为 synchronized |
|   ACC_VARARGS    |   声明方法有变长参数    |
|   ACC_ABSTRACT   |   声明方法为 abstract   |
