---
title: C语言 函数指针 & 指针函数
date: 2018-12-17 13:35:00
tags:
- C
- Function Pointer
- Pointer Function
- 函数指针
- 指针函数
---

C语言中，指针函数（Pointer Function）与函数指针（Function Pointer）

**指针函数**=返回指针的函数：`int *func(int a, int b);`
> 1. 声明的是一个**函数**；
2. 函数的**返回类型是指针**

**函数指针**=指向函数的指针：`void (*func)(int a, int b);`
> 1. 声明的是一个**指针**；
2. 一般指针：指向一个变量的内存地址；
函数指针：指向一个函数的首地址；

```
-- 函数指针的使用方式
// 定义一个函数指针 func
void (*func)(int a, int b);
// 声明一个函数原型 add
void add(int x, int y);
// 为函数指针赋值：将add()函数首地址赋值给func指针
func = add;
```
