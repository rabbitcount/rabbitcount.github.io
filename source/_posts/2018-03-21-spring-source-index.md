---
title: Spring 源码一小堆儿
date: 2018-03-21 22:37:29
tags:
- spring
---

# 获取`ApplicationContext`的两种方式
- 实现 `ApplicationContextAware` 接口
- 通过 `@Autowired` 或 `@Resource` 注解标注让Spring进行自动注入