---
title: Nginx 安装
author: zhangzangqian
date: 2022-12-11 18:00:00 +0800
categories: [技术]
tags: [Nginx, Linux]
math: true
mermaid: true
---

## yum 安装

```bash
# 安装
yum install epel-release -y
yum install nginx -y

# 启动
nginx
```

## 源码编译安装

### 安装

```bash
# PCRE - 支持正则表达式。NGINX Core和Rewrite模块要求。
wget github.com/PCRE2Project/pcre2/releases/download/pcre2-10.40/pcre2-10.40.tar.gz
tar -zxf pcre2-10.40.tar.gz
cd pcre2-10.40
./configure
make && make install

# zlib - 支持标头压缩。NGINX Gzip模块要求。
wget http://zlib.net/zlib-1.2.13.tar.gz
tar -zxf zlib-1.2.13.tar.gz
cd zlib-1.2.13
./configure
make && make install

# OpenSSL - 支持HTTPS协议。NGINX SSL模块和其他模块要求。
wget http://www.openssl.org/source/openssl-1.1.1p.tar.gz
tar -zxf openssl-1.1.1p.tar.gz
cd openssl-1.1.1p
./Configure darwin64-x86_64-cc --prefix=/usr
make && make install

# 正式安装 nginx
wget https://nginx.org/download/nginx-1.23.2.tar.gz

tar zxf nginx-1.23.2.tar.gz

cd nginx-1.23.2

./configure
--prefix=/usr/local/nginx
--with-pcre=../pcre2-10.40
--with-zlib=../zlib-1.2.13
--with-http_ssl_module
--with-stream
--with-threads

make && make install

# 启动
nginx
```

安装参考: <https://docs.nginx.com/nginx/admin-guide/installing-nginx/installing-nginx-open-source>

### 设置开机自启

1. 通过 `vim /lib/systemed/system/nginx.service` 新建 `nginx.service` 文件，并添加如下内容

    ```
    [Unit]
    Description=nginx
    After=network.target

    [Service]
    Type=forking
    ExecStart=/usr/local/nginx/sbin/nginx
    ExecReload=/bin/kill -s HUP $MAINPID
    ExecStop=/bin/kill -s QUIT $MAINPID
    PrivateTmp=true

    [Install]
    WantedBy=multi-user.target
    ```
    {: file='nginx.service'}

    文件解释

    ```
    [Unit]:服务的说明
    Description:描述服务
    After:描述服务类别

    [Service]服务运行参数的设置
    Type=forking是后台运行的形式
    ExecStart为服务的具体运行命令
    ExecReload为重启命令
    ExecStop为停止命令
    PrivateTmp=True表示给服务分配独立的临时空间
    注意：启动、重启、停止命令全部要求使用绝对路径

    [Install]服务安装的相关设置，可设置为多用户
    ```
    {: file='xxx.service'}

2. 设置权限并使文件生效

    ```bash
    sudo chmod +x /lib/systemed/system/nginx.service
    systemctl daemon-reload
    ```
3. 设置开机自启

    ```bash
    systemctl enable nginx
    ```

    只有返回类似 `Created symlink from /etc/systemd/system/multi-user.target.wants/nginx.service to /etc/systemd/system/nginx.service` 提示才能够确定设置成功。如果不是请检查上面的文件

4. 其他命令

    ```bash
    systemctl is-enabled servicename.service #查询服务是否开机启动
    systemctl enable *.service #开机运行服务
    systemctl disable *.service #取消开机运行
    systemctl start *.service #启动服务
    systemctl stop *.service #停止服务
    systemctl restart *.service #重启服务
    systemctl reload *.service #重新加载服务配置文件
    systemctl status *.service #查询服务运行状态
    systemctl --failed #显示启动失败的服务
    ```
