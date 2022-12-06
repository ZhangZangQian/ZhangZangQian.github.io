---
title: Neo4j 学习之安装
author: zhangzangqian
date: 2022-11-19 10:50:00 +0800
categories: [技术]
tags: [Neo4j, DB]
math: true
mermaid: true
---



## 本地安装（Linux环境）
1. 安装 JDK 11，因为我本地之前已经安装了 Java 8，因此推荐使用 Jenv 进行 Java 版本管理
2. 执行如下命令
	
	```bash
	# 导入 neo4j 的 yum 源
	rpm --import https://debian.neo4j.com/neotechnology.gpg.key

	# 编辑 neo4j 源配置文件，添加如下内容
	vim /etc/yum.repos.d/neo4j.repo

	[neo4j]
	name=Neo4j RPM Repository
	baseurl=https://yum.neo4j.com/stable/4.4
	enabled=1
	gpgcheck=1

	# 安装 neo4j
	yum install neo4j-4.4.9

	# 启动
	systemctl start neo4j

	# 设置开机自启
	systemctl enable neo4j
	```
3. 如此 neo4j 便安装成功了，此时 neo4j 是无法被远程访问的，需要修改配置，取消 `/etc/neo4j/neo4j.conf` 文件中对 `dbms.default_listen_address=0.0.0.0`的注释，然后重启`systemctl restart neo4j`。

## Docker 安装

```bash
# 1. 拉取 docker 镜像
docker pull neo4j

# 2. 启动容器
docker run -d --name neo4j \  //-d表示容器后台运行 --name指定容器名字
	-p 7474:7474 -p 7687:7687 \  //映射容器的端口号到宿主机的端口号
	-v /home/neo4j/data:/data \  //把容器内的数据目录挂载到宿主机的对应目录下
	-v /home/neo4j/logs:/logs \  //挂载日志目录
	-v /home/neo4j/conf:/var/lib/neo4j/conf   //挂载配置目录
	-v /home/neo4j/import:/var/lib/neo4j/import \  //挂载数据导入目录
	--env NEO4J_AUTH={username}/{password} \  //设定数据库的名字的访问密码
	neo4j //指定使用的镜像
```

参考：<https://neo4j.com/docs/operations-manual/4.4/installation/linux/rpm/>