---
title: Centos7 安装 Confluence 7.19.5 并破解
author: zhangzangqian
date: 2023-02-01 16:00:00 +0800
categories: [技术]
tags: [Linux, Confluence]
math: true
mermaid: true
---

> 安装 confulence 之前需要安装 mysql，参考 [CentOS7 安装 MySQL8](/posts/mysql-install/){: target="blank"}
{: .prompt-tip}

## 前置准备

### 安装 Java

此处不做过多赘述，网上各种教程都有

###  下载软件

```bash
# confluence 安装包
wget https://www.atlassian.com/software/confluence/downloads/binary/atlassian-confluence-7.19.5-x64.bin
# 破解软件
wget https://github.com/haxqer/confluence/releases/download/v1.3.3/atlassian-agent.jar
```

## 安装并运行

### 配置 MySQL

1. 建库

    ```sql
    create database confluence character set utf8mb4 collate utf8mb4_bin;
    ```

2. 修改 `/etc/my.cnf` 文件，并在 `[mysqld]` 下添加如下内容后执行 `systemctl restart mysqld` 重启 MySQL。

    ```console
    character-set-server=utf8mb4 
    collation-server=utf8mb4_bin
    default-storage-engine=INNODB
    max_allowed_packet=256M 
    innodb_log_file_size=2GB
    transaction-isolation=READ-COMMITTED
    binlog_format=row
    log-bin-trust-function-creators = 1
    ```

### 安装

```bash
# 添加可执行权限
chmod u+x atlassian-confluence-7.19.5-x64.bin
# 执行安装脚本，然后一路回车键，不需要修改任何东西
./atlassian-confluence-7.19.5-x64.bin
```

### 修改配置并启动服务

1. 修改 `/opt/atlassian/confluence/bin/setenv.sh` 文件，找到 `CATALINA_OPTS` 并新增一行 `CATALINA_OPTS="-javaagent:/path/to/atlassian-agent.jar ${CATALINA_OPTS}"`，截图如下:
    
    ![setenv.sh](/assets/img/c-i-c-1.png)

2. 配置 MySQL 链接驱动
    
    ```bash
    wget https://repo1.maven.org/maven2/com/mysql/mysql-connector-j/8.0.31/mysql-connector-j-8.0.31.jar -P /opt/atlassian/confluence/lib/
    ```

3. 执行 `service confluence start` 启动 confluence。

## 初始化&破解

此时我们访问 `http://{ip}:8090` 即可访问系统了(**确保防火墙开放了 8090 端口**)，然后根据下图操作:

![confluence 安装引导页](/assets/img/c-i-c-2.png)

下一步后来到此处

![confluence 秘钥验证页](/assets/img/c-i-c-3.png)

复制**服务器ID**，然后在服务器执行 `java -jar atlassian-agent.jar -p conf -m ${你的邮箱地址} -n ${你的邮箱地址} -o http://${ip}:8090 -s ${服务器 ID}` 命令，得到类似如下结果，复制秘钥到系统，然后下一步即可（可能会等一段时间）。

![生成秘钥结果](/assets/img/c-i-c-4.png)

然后设置数据，选择 “我自己的数据库” 然后下一步

![选择数据库](/assets/img/c-i-c-5.png)

选择 MySQL，并填入链接信息并下一步（此步操作可能又要等一会。）

![配置数据库链接](/assets/img/c-i-c-6.png)

根据需要选择，我们此处选择 "示范站点"

![加载内容页](/assets/img/c-i-c-7.png)

配置管理员账户后下一步

![加载内容页](/assets/img/c-i-c-8.png)

最后 confluence 便是安装完成了，后续根据指引操作即可。

## 添加 nginx 反向代理

1. 安装nginx，参考[Nginx 安装](/posts/nginx-install/){: target="blank"}

2. 配置 nginx，添加 server，配置如下

    ```conf
    server {
        listen       80;
        listen       [::]:80;
        server_name  你的域名;

        # Load configuration files for the default server block.

        location / {
            proxy_set_header X-Forwarded-Host $host;
                proxy_set_header X-Forwarded-Server $host;
                proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for; 
                proxy_pass http://ip:8090/;
                client_max_body_size 1024M;
        }	
    }
    ```
3. 修改 confluence 的 server.xml 配置，在 `/opt/atlassian/confluence/conf` 目录下，按下如修改，proxyName 替换为你的域名

    ![修改 server.xml 配置](/assets/img/c-i-c-9.png)

4. 执行 `service confluence restart` 重启 confluence 并通过域名访问即可。