---
title: CentOS 7.9 搭建 Jenkins 服务
author: zhangzangqian
date: 2022-10-07 12:00:00 +0800
categories: [技术]
tags: [Linux, Jenkins]
math: true
mermaid: true
---

## 修改 yum 源

```bash
# 修改源为清华源
sudo sed -e 's|^mirrorlist=|#mirrorlist=|g' \
         -e 's|^#baseurl=http://mirror.centos.org|baseurl=https://mirrors.tuna.tsinghua.edu.cn|g' \
         -i.bak \
         /etc/yum.repos.d/CentOS-*.repo
# 更新软件包缓存
sudo yum makecache
```

## 安装 docker

```bash
sudo yum remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-engine
sudo yum install -y yum-utils

sudo yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo
    
sudo yum install docker-ce docker-ce-cli containerd.io docker-compose-plugin

systemctl start docker

# 安装 docker-compose
DOCKER_CONFIG=${DOCKER_CONFIG:-$HOME/.docker}
mkdir -p $DOCKER_CONFIG/cli-plugins
curl -SL https://github.com/docker/compose/releases/download/v2.11.2/docker-compose-linux-x86_64 -o $DOCKER_CONFIG/cli-plugins/docker-compose
```

## docker 安装 Jenkins

```bash
# 拉取jenkins镜像
docker pull jenkins/jenkins:lts
# 创建数据目录
mkidr -p /data/jenkins_home

# 启动镜像
docker run -p 8613:8080 -p 18613:5000 --name jenkins -u root -v /data/jenkins_home:/var/jenkins_home -d jenkins/jenkins:lts
```

## Jenkins 初始化

如此便将 Jenkins 安装完成了，现在我们可以通过浏览器进行访问：http://ip:8613/，**如果无法访问请检查防火墙是否开放了端口，如果是云服务器还要检查安全组配置。**

![Jenkins 初始化页面](/assets/img/c-jk-1.png)

首次启动jenkins需要输入密码，需要进入容器内获取密码。**密码位于/var/jenkins_home/secrets/initialAdminPassword。因为我们设置了卷宗映射，所以我们无需进入 docker 容器，直接在宿主机就可以查看。**


```bash
cat /data/jenkins_home/secrets/initialAdminPassword
68eed23ad39541949972468e4f2ce1fd
```

输入密码以后，安装需要的插件，在安装途中由于网络原因会出现有些插件安装失败，这个可以不用理会。

![Jenkins 初始化页面](/assets/img/c-jk-2.png)
![Jenkins 初始化页面](/assets/img/c-jk-3.png)
![Jenkins 初始化页面](/assets/img/c-jk-4.png)
![Jenkins 初始化页面](/assets/img/c-jk-5.png)

设置Jenkins的默认登录账号和密码

![Jenkins 初始化页面](/assets/img/c-jk-6.png)
![Jenkins 初始化页面](/assets/img/c-jk-7.png)

## 遇到的问题

### 插件下载慢

修改 `{jenkins_home}/hudson.model.UpdateCenter.xml` 文件

```xml
<?xml version='1.1' encoding='UTF-8'?>
<sites>
  <site>
    <id>default</id>
    <url>https://updates.jenkins.io/update-center.json</url>
  </site>
</sites>
<!-- 将下载地址替换为http://mirror.esuni.jp/jenkins/updates/update-center.json -->
<?xml version='1.1' encoding='UTF-8'?>
<sites>
  <site>
    <id>default</id>
    <url>http://mirror.esuni.jp/jenkins/updates/update-center.json</url>
  </site>
</sites>
```

### 报错

![Jenkins 报错](/assets/img/jk-error-1.png){: .w-50}

升级 Jenkins 进行解决

#### 自动升级

![Jenkins 初始化页面](/assets/img/c-jk-8.png)

#### 手动升级

可以去Jenkins的官网下载好最新jar包上传到服务器，也可以使用wget命令。

```bash
# wget http://jenkins新版本的下载地址，目前最新 2.370
wget https://mirrors.tuna.tsinghua.edu.cn/jenkins/war/2.370/jenkins.war --no-check-certificate
```


Jenkins 的更新主要是替换 Jenkins 镜像里面的 war 包 ，我们可以把下载好的 war 包使用 docker cp 直接进行复制命令如下：

```bash
docker cp jenkins.war jenkins:/usr/share/jenkins
```

重新启动 Jenkins 即可完成升级。

```bash
docker restart jenkins
```