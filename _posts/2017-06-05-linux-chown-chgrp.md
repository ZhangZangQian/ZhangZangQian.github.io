---
title: Linux 修改文件所属用户和组
author: zhangzangqian
date: 2017-06-05 19:00:00 +0800
categories: [技术]
tags: [Linux]
math: true
mermaid: true
---

## 使用chown命令可以修改文件或目录所属的用户

命令：chown 用户 目录或文件名，例子如下：

```bash
# 把home目录下的qq目录的拥有者改为qq用户
chown qq /home/qq
```

## 使用chgrp命令可以修改文件或目录所属的组：

命令：chgrp 组 目录或文件名，例子如下：

```bash
# 把home目录下的qq目录的所属组改为qq组
chgrp qq /home/qq
```

## 同时切换文件所属组和用户

```bash
chown user:group /home/qq
```