---
title: 配置 github ssh 加速
author: zhangzangqian
date: 2022-12-15 18:00:00 +0800
categories: [技术]
tags: [git]
---

以前 github 使用 ssh 协议 fetch 和 push 一直没有问题，今天 push 代码时不知道为什么报错了。问题如下：


> kex_exchange_identification: Connection closed by remote host<br/>Connection closed by 127.0.0.1 port 7890<br/>fatal: Could not read from remote repository.<br/>Please make sure you have the correct access rights and the repository exists.
{: .prompt-danger}

通过执行 `ssh -T git@github.com` 命令发现在建立建立的步骤停住走不下去了，如下图

![ssh -T 测试](/assets/img/git-ssh-T.png)

所以尝试为 git 配置一个代理（可以翻墙的），所以修改 `~/.ssh/config` 文件添加如下内容

```config
 42 Host github.com
 48   # 走 socks5 代理（如 Shadowsocks）
 49   ProxyCommand nc -v -x 127.0.0.1:7890 %h %p
```
{: file='~/.ssh/config'}

发现连接可以建立成功了，但还是报错，内容如下 ：

> Connection to github.com port 22 [tcp/ssh] succeeded!<br/>kex_exchange_identification: Connection closed by remote host<br/>Connection closed by UNKNOWN port 65535
{: .prompt-danger}

于是通过查询资料，完善了下配置，成功 push 上去了代码，完整配置内容如下

```config
 42 Host github.com
 43   User git
 44   HostName ssh.github.com
 45   Port 443
 46   # 走 HTTP 代理， ubuntu 先 apt install -y socat
 47   #ProxyCommand socat - PROXY:127.0.0.1:%h:%p,proxyport=1080
 48   # 走 socks5 代理（如 Shadowsocks）
 49   ProxyCommand nc -v -x 127.0.0.1:7890 %h %p
```
{: file='~/.ssh/config'}

注：本人是 mac 系统，Windows 系统下未进行测试。

参考链接：<https://gist.github.com/chenshengzhi/07e5177b1d97587d5ca0acc0487ad677?permalink_comment_id=3721523>