---
title: Java 字节码
date: 2019-01-12 21:48:55
tags:
- jvm
---

JVM 字节码定义与解析

<!-- more -->

# 字节码一览

# 解析字节码

## 解析常量池主要链路
```C++
ClassFileParser::parseClassFile
  ┗━ ClassFileParser::parse_constant_pool()
      ┗━ oopFactory::new_constantPool()                  // 为常量池分配内存
      ┗━ ClassFileParser::parse_constant_pool_entries()  // 解析常量池
```

### 常量池内存分配
JVM 解析 constant_pool 部分的信息前，需要先划出一块内存空间，将常量池的结构信息加载进来；会涉及两个问题：
1. 分配多少内存
2. 分配在哪里

JVM 使用专门的 C++ 类 constantPoolOop 保存常量池信息
```C++
class oopDesc {
  private:
    volatile markOop _mark;  // 线程锁等标记
    union _metadata {
      wideKlassOop  _klass;
      narrowOop     _compressed_klass;
    } _metadata;             // 元数据，自引用
}
```