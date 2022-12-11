---
title: 银河麒麟 MySQL 启动
author: zhangzangqian
date: 2022-12-11 17:40:00 +0800
categories: [技术]
tags: [MySQL, DB]
---

一个军工项目，使用的银河麒麟的服务器，没有外网，登录进去之后发现 MySQL 已经安装了，但是没有启动，于是尝试使用 `mysqld` 命令启动，发现根本没有 `mysqld` 命令。通过 `mysql -V` 命令查看发现是 `MariaDB 10.3.9` 版本。

于是尝试使用 `systemctl start mariadb.service` 启动，发现果然成功了。然后通过 `systemctl enable mariadb.service` 设置开机自启。