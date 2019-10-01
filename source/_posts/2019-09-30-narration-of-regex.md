title: 大话正则表达式
date: 2019-09-30T07:17:24.299Z
tags: [regex]
categories: []
---
# 语法

## 元字符

| 字符 | 描述 |
| --- | --- |
| \\d | 匹配一个数字字符。等价于 \[0\-9\]。 |
| \\D | 匹配一个非数字字符。等价于 \[^0\-9\]。 |
| \\w | 匹配字母、数字、下划线。等价于'\[A\-Za\-z0\-9\_\]' |
| \\W | 匹配非字母、数字、下划线。等价于 '\[^A\-Za\-z0\-9\_\]' |
| \\s | 匹配任何空白字符，包括空格、制表符、换页符 |
| \\S | 匹配任何非空白字符。等价于 \[^ \\f\\n\\r\\t\\v\] |
| . | 匹配除换行符（\\n、\\r）之外的任何单个字符。要匹配包括 '\\n' 在内的任何字符，请使用像"(. |
| \\f | 匹配一个换页符。 |
| \\n | 匹配一个换行符。 |
| \\r | 匹配一个回车符。 |
| \\t | 匹配一个制表符。 |
| \\v | 匹配一个垂直制表符。 |
| ^ | 匹配输入字符串开始的位置。 |
| $ | 匹配输入字符串结尾的位置 |
| \\b | 匹配一个单词边界，也就是指单词和空格间的位置。例如， 'er\\b' 可以匹配"never" 中的 'er'，但不能匹配 "verb" 中的 'er'。 |
| \\B | 与 \\b 相反：er\\B' 能匹配 "verb" 中的 'er'，但不能匹配 "never" 中的 'er'。 |

## 区间

| 字符 | 描述 |
| --- | --- |
| \[0\-9\] | 匹配 0\-9 之间的数字 |
| \[A\-Z\] | 匹配 A\-Z 之间的字母，也可以组合 \[A\-Za\-z0\-9\] |

## 限定符

| 字符 | 描述 |
| --- | --- |
| \* | 匹配前面的子表达式零次或多次。例如，zo\* 能匹配 "z" 以及 "zoo"。\* 等价于{0,} |
| + | 匹配前面的子表达式一次或多次。例如，'zo+' 能匹配 "zo" 以及 "zoo"，但不能匹配 "z"。+ 等价于 {1,} |
| ? | 匹配前面的子表达式零次或一次。例如，"do(es)?" 可以匹配 "do" 、 "does" 中的 "does" 、 "doxy" 中的 "do" 。? 等价于 {0,1} |
| {n} | n 是一个非负整数。匹配确定的 n 次。例如，'o{2}' 不能匹配 "Bob" 中的 'o'，但是能匹配 "food" 中的两个 o |
| {n,} | n 是一个非负整数。至少匹配n 次。例如，'o{2,}' 不能匹配 "Bob" 中的 'o'，但能匹配 "foooood" 中的所有 o。'o{1,}' 等价于 'o+'。'o{0,}' 则等价于 'o\*' |
| {n,m} | m 和 n 均为非负整数，其中n <= m。最少匹配 n 次且最多匹配 m 次。例如，"o{1,3}" 将匹配 "fooooood" 中的前三个 o。'o{0,1}' 等价于 'o?'。请注意在逗号和两个数之间不能有空格 |

## 子表达式

用圆括号组成一个比较复杂的匹配模式，那么一个圆括号的部分可以看作是一个子表达式。

## 捕获 & 反捕获

- 多个子表达式所匹配到的内容按顺序出现在内存的缓冲区中`捕获数组`，称为捕获；
- 反捕获 与 捕获相反，标记不需要捕获的内容；

## 反向引用

圆括号的内容被捕获后，可以在这个括号后被使用，从而写出一个比较实用的匹配模式；

## 贪婪匹配

当正则表达式中包含能接受重复的限定符时，通常的行为是（在使整个表达式能得到匹配的前提下）匹配尽可能多的字符；

## 懒惰 / 非贪婪

当正则表达式中包含能接受重复的限定符时，通常的行为是（在使整个表达式能得到匹配的前提下）匹配尽可能少的字符；

懒惰量词是在贪婪量词后面加个`?`

| 代码 | 说明 |
| --- | --- |
| \*? | 重复多次，但尽可能少重复 |
| +? | 重复1次、多次，但尽可能少重复 |
| ?? | 重复0次、1次，但尽可能少重复 |
| {n,m}? | 重复n~m次，但尽可能少重复 |
| {n,}? | 重复n次以上，但尽可能少重复 |


