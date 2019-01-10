---
title: Python vs Java
date: 2017-05-30 20:00:44
tags:
- python
- java
- vs
---

# 静态语言 和 动态语言

---
**鸭子类型**`并不要求严格的继承体系，一个对象只要“看起来像鸭子，走起路来像鸭子”，那它就可以被看做是鸭子。`

---


> 静态语言Java，如果需要传入Animal类型，则传入的对象必须是Animal类型或者它的子类。
> 动态语言python，不一定传入Animal类型，只需要保证其中有run方法即可
```
class Animal(object):
    def run(self):
        print('Animal is running...')

def run_twice(animal):
    animal.run()
    animal.run()

class Timer(object):
    def run(self):
        print('Start...')

```