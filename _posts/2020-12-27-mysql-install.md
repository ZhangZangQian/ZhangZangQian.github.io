---
title: CentOS7 安装 MySQL8
author: zhangzangqian
date: 2020-12-27 21:00:00 +0800
categories: [技术]
tags: [Linux, MySQL, DB]
math: true
mermaid: true
---

## 在线安装

```bash
# 下载 rpm 包
wget  https://repo.mysql.com//mysql80-community-release-el7-1.noarch.rpm
# 安装 yum repo文件并更新 yum 缓存
rpm -ivh mysql80-community-release-el7-1.noarch.rpm
yum clean all
yum makecache
# yum 安装 mysql
yum install mysql-community-server -y --nogpgcheck
# 启动 mysql
systemctl start mysqld
# 获取 mysql 密码
cat /var/log/mysqld.log | grep password
# 使用初始密码登录 mysql
mysql -uroot -p
```

执行 sql 修改密码

```sql
# 注意位数和种类至少大+写+小写+符号+数字
ALTER USER 'root'@'localhost' IDENTIFIED BY 'MyNewPass4!';
```

## 离线安装

### 下载 rpm 包

首先在一台联网电脑下载好安装包

```bash
wget https://dev.mysql.com/get/Downloads/MySQL-8.0/mysql-8.0.31-1.el7.x86_64.rpm-bundle.tar

tar xvf mysql-8.0.31-1.el7.x86_64.rpm-bundle.tar
```

### 上传至离线服务器并解压

上传至需要安装的服务器，解压得到如下安装包:

- mysql-community-client-8.0.31-1.el7.x86_64.rpm
- mysql-community-client-plugins-8.0.31-1.el7.x86_64.rpm
- mysql-community-common-8.0.31-1.el7.x86_64.rpm
- mysql-community-debuginfo-8.0.31-1.el7.x86_64.rpm
- mysql-community-devel-8.0.31-1.el7.x86_64.rpm
- mysql-community-embedded-compat-8.0.31-1.el7.x86_64.rp
- mysql-community-icu-data-files-8.0.31-1.el7.x86_64.rpm
- mysql-community-libs-8.0.31-1.el7.x86_64.rpm
- mysql-community-libs-compat-8.0.31-1.el7.x86_64.rpm
- mysql-community-server-8.0.31-1.el7.x86_64.rpm
- mysql-community-server-debug-8.0.31-1.el7.x86_64.rpm
- mysql-community-test-8.0.31-1.el7.x86_64.rpm

### 安装

依次安装：

```bash
rpm -ivh mysql-community-common-8.0.31-1.el7.x86_64.rpm
rpm -ivh mysql-community-client-plugins-8.0.31-1.el7.x86_64.rpm
rpm -ivh mysql-community-libs-8.0.31-1.el7.x86_64.rpm
rpm -ivh mysql-community-client-8.0.31-1.el7.x86_64.rpm
rpm -ivh mysql-community-icu-data-files-8.0.31-1.el7.x86_64.rpm
rpm -ivh mysql-community-server-8.0.31-1.el7.x86_64.rpm
```

1. 安装 `mysql-community-libs-8.0.31-1.el7.x86_64.rpm` 时可能会得到 `mysql-community-libs-8.0.31-1.el7.x86_64.rpm: Header V4 RSA/SHA256 Signature xxx : NOKEY` 报错，执行 `yum remove mysql-libs` 即可解决。
2. 安装 `mysql-community-server-8.0.31-1.el7.x86_64.rpm` 时可能会得到 `libaio.so.1()(64bit) is needed by mysql-community-server-8.0.31-1.el7.x86_64` 类似报错，执行 `yum install libaio` 即可解决。

### 运行

```bash
# 初始化 MySQL 服务
mysqld --initialize --user=mysql
# 查看初始化密码
cat /var/log/mysqld.log | grep 'temporary password'
# 启动 MySQL
systemctl start mysqld
# 登录
mysql -uroot -p
```

登录成功后需要更改初始化密码后方可进行正常操作，执行如下 SQL 修改密码:

```sql
alter user 'root'@'localhost' identified by '新密码';

show databases;
```

如此 MySQL 安装完成！