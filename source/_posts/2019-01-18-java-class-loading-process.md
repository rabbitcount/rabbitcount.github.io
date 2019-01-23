---
title: Java 类加载过程
date: 2019-01-18 15:54:21
tags:
- java
---
Java 类加载过程 = `Loading` + `Linking` + `Initialization`
<!-- more --> 

# Specification
The Java Virtual Machine dynamically loads, links and initializes classes and interfaces. 
- `Loading`
 is the process of finding the binary representation of a class or interface type with a particular name and creating a class or interface from that binary representation. 
- `Linking` 
 is the process of taking a class or interface and combining it into the run-time state of the Java Virtual Machine so that it can be executed. 
- `Initialization`
 of a class or interface consists of executing the class or interface initialization method `<clinit>`
 
![](https://wiki-1258407249.cos.ap-chengdu.myqcloud.com/2019-01-18-java-class-loading-process/jvm-class-loader-loading-process.png)

## Creation and Loading 加载
> If **Class C** is not an array class, it is created by loading a binary representation of **C**  using a class loader. 
> Array classes do not have an external binary representation; they are created by the Java Virtual Machine rather than by a class loader.

- 通过一个类的全限定名获取其定义的二进制字节流；
- 将这个字节流所代表的静态存储结构转化为方法区的运行时数据结构；
- 在堆中生成一个代表这个类的 `java.lang.Class` 对象，作为对方法区中这些数据的访问入口；

## Linking

### Verification 验证
**加载和验证是交叉进行的，验证在各个阶段都是存在的**
- 确保Class文件的字节流信息符合JVM的要求；
- 4个阶段校验(文件格式校验、元数据校验、字节码校验、符号引用校验)；
- 验证阶段是非常重要的，但不是必须的，它对程序运行期没有影响，如果所引用的类经过反复验证，那么可以考虑采用 `-Xverifynone` 参数来关闭大部分的类验证措施，以缩短虚拟机类加载的时间。

### Preparation 准备
- 为类的静态变量(static)分配内存，并将其初始化为默认值；
- 如果常量无初始值，则默认赋值为java基本数据类型的默认值

| 类型           | 描述                 |
| ------------- |:-------------------:|
|byte |用8位补码表示，初始化为0|
|short|用16位补码表示，初始化为0|
|int|用32位补码表示，初始化为0|
|long|用64位补码表示，初始化为0L|
|char|用16位补码表示，初始化为”u0000”，使用UTF-16编码|
|float|初始化为正0|
|double|初始化为正0|
|boolean|初始化为0|
|returnAddress|初始化为字节码指令的地址,用于配合异常处理特殊指令|

### Resolution 解析
解析阶段是虚拟机将常量池内的符号引用替换为直接引用的过程
- **符号引用（Symbolic References）**
符号引用以一组符号来描述所引用的目标，符号可以是任何形式的字面量，只要使用时能无歧义地定位到目标即可。
- **直接引用（Direct References）**
直接引用可以是直接指向目标的指针、相对偏移量或是一个能间接定位到目标的句柄。如果有了直接引用，那么引用的目标一定是已经存在于内存中。

这一步是**可选**的。可以在符号引用第一次被使用时完成，即所谓的 **延迟解析(late resolution)** 。但对用户而言，这一步永远是延迟解析的，即使运行时会执行 early resolution，但程序不会显示的在第一次判断出错误时抛出错误，而会在对应的类第一次主动使用的时候抛出错误！

## Initialization 初始化
- 对类的静态变量，静态代码块执行初始化操作

类的初始化也是延迟的，直到类第一次被主动使用(active use)，JVM 才会初始化类；

类的初始化分两步：
- 如果基类没有被初始化，初始化基类。
- 有类构造函数，则执行类构造函数。

类构造函数是由 Java 编译器完成的。它把类成员变量的初始化和 static 区间的代码提取出，放到一个的方法中。这个方法不能被一般的方法访问（注意，static final 成员变量不会在此执行初始化，它一般被编译器生成 constant 值）。同时，中是不会显示的调用基类的的，因为 1 中已经执行了基类的初始化。类的初始化还必须注意线程安全的问题。
