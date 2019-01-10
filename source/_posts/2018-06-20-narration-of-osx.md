---
title: OSX的那些事儿
date: 2018-06-20 15:18:44
tags:
- osx
- shell
---

# Bash

`~/.bash_profile`
`~/.bashrc`
`~/.bash_logout`

`.bash_profile`: 每次用户登录系统时读取，其中的所有命令都会被bash执行
`.bashrc`: 在shell中键入bash命令启动一个新的shell时读取
`.bash_logout`: 退出shell时被读取

## Shell类型
是否为登录Shell（login shell），是否为交互式Shell（interactive shell）

- 交互式Shell：
没有非选项参数，没有`-c`，标准输入和标准输出与终端相连的，或者用`-i`参数启动的Bash实例。可以通过探测PS1变量是否设置或者$-返回值中是否包含字幕i来判定。
什么是没有非选项参数？比如bash ~/myscript/clear_temp_files.sh这样执行的Shell脚本的Bash实例，就不是交互式Shell，因为脚本的路径，就是非选项参数。
`-c`又是干什么的？就是使用一个字符串作为Bash的传入参数，比如bash -c ‘ls -ahl’，这种Shell进程也不算是交互式Shell。
- 登录Shell：
第0个参数以-号开头的Bash实例（登录后，执行`echo $0`可见 `-` 开头），或者用-login参数启动的Bash Shell
**非登录Shell的用途**比如一个用Linux搭建一个ftp服务器，并且创建了很多的ftp用户，那么就可以将这些用户的默认shell改为nologin，这样一来，这些虽然是Linux上的用户可是却无法登录进Linux主机，只能登录ftp服务器了，保证了安全性。

登录shell时文件的加载顺序

- `/etc/profile`
- `~/.bash_profile`
- `~/.bash_login`
- `~/.profile`

## 安装zsh后

安装了zsh之后默认启动执行脚本变为了`～/.zshrc`
可以考虑在其中添加
```
source ~/.bash_profile
source ~/.bashrc
```

