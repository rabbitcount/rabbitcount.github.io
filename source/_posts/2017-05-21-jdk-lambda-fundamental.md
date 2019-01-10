---
title: lambda基础
date: 2017-05-21 12:34:27
tags: 
 - lambda
 - jdk
 - java
---

# Function接口

`Predicate<T>` --> 接收`T`对象并返回`boolean`（<font color=#0099ff>**断言，返回true、false**</font>）
`Consumer<T>` --> 接收`T`对象，不返回值（<font color=#0099ff>**处理，无返回**</font>）
`Function<T, R>` --> 接收`T`对象，返回`R`对象，T --> R（<font color=#0099ff>**`T`转换为`R`**</font>）
`Supplier<T>` --> 提供`T`对象（例如工厂），不接收值（<font color=#0099ff>**构造器**</font>）
`UnaryOperator<T>` --> 接收`T`对象，返回`T`对象（<font color=#0099ff>**一元运算**</font>）
`BinaryOperator<T>` --> 接收两个`T`对象，返回`T`对象（<font color=#0099ff>**二元运算**</font>）
