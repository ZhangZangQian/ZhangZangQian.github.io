---
title: docker 登录 harbor x509 问题解决
author: zhangzangqian
date: 2023-04-20 11:00:00 +0800
categories: [技术]
tags: [Linux, Docker, Habor]
math: true
mermaid: true
---

当 Docker 客户端尝试连接到私有仓库时，如果出现 x509: certificate signed by unknown authority 错误，通常表示 Docker 客户端没有私有仓库证书的信任链。可以按照以下步骤进行修复：

1. 手动下载证书

    首先，可以使用 openssl 命令手动下载私有仓库的证书。假设私有仓库的 URL 为 https://harbor.example.com，则可以运行以下命令：

    ```bash
    openssl s_client -showcerts -connect harbor.example.com:443 </dev/null 2>/dev/null | openssl x509 -outform PEM > /path/to/harbor.example.com.crt
    ```

    该命令将从 https://harbor.example.com 下载 SSL 证书并将其保存到 /path/to/harbor.example.com.crt 文件中。

2. 将证书添加到 Docker 信任链

    将证书添加到 Docker 的信任链中，使其信任私有仓库的 SSL 证书。可以使用以下命令将证书复制到 Docker 的证书存储目录：

    ```bash
    sudo mkdir -p /etc/docker/certs.d/harbor.example.com/
    sudo cp /path/to/harbor.example.com.crt /etc/docker/certs.d/harbor.example.com/
    ```

3. 重新启动 Docker 守护进程

    重新启动 Docker 守护进程以使其重新加载证书。可以使用以下命令重新启动 Docker：
    ```bash
    sudo systemctl restart docker
    ```

4. 登录 Harbor

    现在，您应该可以登录 Harbor 仓库了。可以使用以下命令登录 Harbor：

    ```bash
    docker login harbor.example.com
    ```
    输入您的用户名和密码即可成功登录。