---
title: Linux 使用 ssh key 进行免密登录
author: zhangzangqian
date: 2017-06-10 19:00:00 +0800
categories: [技术]
tags: [Linux]
math: true
mermaid: true
---

> 举例服务器 A 使用 ssh key 登录服务器 B。
{: .prompt-tip}

## 免密登录设置

在服务器 A 执行如下命令

```bash
# 执行命令然后连按3次回车，ssh key 创建完毕
ssh-keygen -t rsa
# username 和 ip 均为 服务器 B 信息
ssh-copy-id {username}@{ip}
```

> 使用以上操作便可以实现通过 ssh key 来进行免密远程登录了。以下讲的是设置用户如何进行免密 sudo 操作，禁止密码登录以及禁止 root 账户远程登录。

## 使用 root 账户设置允许指定用户进行免密 sudo 操作

假设指定账户为 test，使用 `vim /ect/sudoers` 编辑 sudoers 文件，在 `root ALL=(ALL) ALL` 下增加 `test ALL=(ALL) ALL` 即可允许用户进行 sudo 操作，在 `# %wheel ALL=(ALL) NOPASSWD: ALL` 下增加 `test ALL=(ALL) NOPASSWD:ALL` 即可实现免密 sudo 操作。

## 禁止密码登录以及 root 账号远程登录

- 禁止密码登录
    
    修改 `/ect/ssh/sshd_confg` 文件，将 `PasswordAuthentication` 设置为 `no`

- 禁止 root 账号远程登录
    
    修改 `/etc/ssh/sshd_confg` 文件，将 `PermitRootLogin` 设置为 `no`

修改完毕后使用 `sudo service sshd restart` 重启 ssh 服务即可。