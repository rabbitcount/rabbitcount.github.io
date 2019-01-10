---
title: CentOS命令行下安装VMware tools
date: 2018-02-25 15:57:33
tags:
- VMware Funsion
- VMware tools
---

# 依赖

- perl
- gcc
- kernel
- net-tools(需要其中的ifconfig)

# 命令行安装VMware tools

- 虚拟光驱
```
mkdir /mnt/cdrom
mount -t iso9660 /dev/cdrom /mnt/cdrom
```

- 拷贝VMWarexxx.tar.gz到/tmp：
```
cp /mnt/cdrom/VMWarexxx.tar.gz /tmp
```

- 解压
```
cd /tmp
tar zxf VMWarexxx.tar.gz
```

- 安装执行
```
cd vmware-tools-disxx
./vmware-install.pl
```
