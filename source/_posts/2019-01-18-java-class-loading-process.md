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

## Creation and Loading
加载
> If **Class C** is not an array class, it is created by loading a binary representation of **C**  using a class loader. 
> Array classes do not have an external binary representation; they are created by the Java Virtual Machine rather than by a class loader.

## Linking

### Verification
验证
### Preparation
准备
### Resolution
解析

## Initialization
初始化