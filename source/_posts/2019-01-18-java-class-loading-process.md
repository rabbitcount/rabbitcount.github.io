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
- 确保Class文件的字节流信息符合JVM的要求；
- 4个阶段校验(文件格式校验、元数据校验、字节码校验、符号引用校验)；
- 验证阶段是非常重要的，但不是必须的，它对程序运行期没有影响，如果所引用的类经过反复验证，那么可以考虑采用 `-Xverifynone` 参数来关闭大部分的类验证措施，以缩短虚拟机类加载的时间。

### Preparation 准备
- 为类的静态变量(static)分配内存，并将其初始化为默认值

### Resolution 解析
- 把类中的符号引用转换为直接引用；
- 符号引用就是一组符号来描述目标，可以是任何字面量；
- 直接引用就是直接指向目标的指针、相对偏移量或一个间接定位到目标的句柄；

## Initialization 初始化
- 对类的静态变量，静态代码块执行初始化操作