---
title: MySQL8 搭建主从复制集群
author: zhangzangqian
date: 2021-01-27 21:00:00 +0800
categories: [技术]
tags: [Linux, MySQL, DB]
math: true
mermaid: true
---

## 目的

1. **最主要目的提升数据库层面性能**，主节点负责写操作，从节点负责读操作。
2. 容错，高可用。
3. 数据备份（搭配快照）。

## 原理

主从复制有两种方式：

- 异步复制：**默认方式**，可能会导致主从切换时数据丢失。因为主库是否commit与主从同步流程无关，也不感知。
- 半同步复制：高可用方案，较新mysql版本支持（配合插件），需要至少1个从库（默认1，具体数量可指定）对写入 relay log 进行 ack，主库才会 commit 并把结果返回 client。

### 异步复制

![主从复制架构图](/assets/img/mysql-ms-2.jpeg)

上图为异步复制的架构，主要流程为：

1. master 执行写 SQL，然后 Dump Thread 会将操作记录到 Binlog 文件
2. slave 节点 IO Thread 请 master 节点发送请求读取 Binlog，master 节点会根据偏移量将新的 Binlog 发送给 slave 节点
3. slave 节点的 IO Thread 在收到 Binlog 后会将记录写入到自身的 relay log 中，这就是所谓的中继日志。
4. slave 节点的 SQL Thread 读取 relay log，重新串行执行，得到与主库相同的数据。

### 半同步复制

#### 主要流程

1. 从库在连接主库时，表明自己支持半同步复制。
2. 主库也需支持半同步复制，主库 commit 事务前会阻塞等待至少一个从库写入 relay log 的 ack，直至超时。
3. 如果阻塞等待超时，则主库临时切换回异步同步模式，当至少一个从库的半同步追上进度时，主库再切换至半同步模式。

## 主从同步应用场景

### 普通场景：线上从库异步同步，高可用备份半同步

![普通场景](/assets/img/mysql-ms-3.jpeg)

### 对一致性要求较高的大数据取数需求

大数据取数可能导致从库cpu使用率飙升、ack变慢，可设置半同步所需ack数量为1，正常情况下高可用备份能很快ack，于是主库会commit并返回，而大数据取数复制慢一些也无所谓。这样就不会因为大数据取数ack慢而影响主库和业务了。

![大数据取数](/assets/img/mysql-ms-4.jpeg)

## 实操配置

1. 主节点开启 binlog 功能，编辑 `/etc/my.cnf` 内容如下，然后执行 `systemctl restart mysqld` 重启 MySQL 服务。

    ```conf
    [mysqld]
    # 设置server_id,注意要唯一
    server-id=100
    # 设置 binlog 文件名
    log-bin=mysql-bin
    # binlog 过期时间
    expire-logs-day=7
    # binlog 格式
    binglog-format=MIXED
    ```
    {: file='my.cnf'}

2. 从节点开启 binlog 和 Relay log，同主节点一样修改配置后重启服务。

    ```conf
    [mysqld]
    # 设置server_id,注意要唯一
    server-id=101  
    # 开启二进制日志功能，以备Slave作为其它Slave的Master时使用
    log-bin=mysql-slave-bin   
    # binlog 过期时间
    expire-logs-day=7
    # binlog 格式
    binglog-format=MIXED
    # relay_log配置中继日志
    relay_log=mysql-relay-bin 
    # 需要复制的数据库
    replicate-do-db=test
    ```

3. 在主库创建数据同步用户，授予用户 slave `REPLICATION SLAVE` 权限和 `REPLICATION CLIENT` 权限，用于在主从库之间同步数据。

    ```sql
    CREATE USER 'slave'@'%' IDENTIFIED BY '123456';
    GRANT REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'slave'@'%';  
    ```

    执行 `show master status`，记住 File 和 Position

    ```console
    ***************************[ 1. row ]***************************
    File              | mysql-bin.000001
    Position          | 2917011
    Binlog_Do_DB      | 
    Binlog_Ignore_DB  | 
    Executed_Gtid_Set | 
    ```

4. 登入从库，执行如下 SQL 配置主库信息：

    ```sql
    change master to master_host='172.16.48.214', master_user='slave', master_password='123456', master_port=3306, master_log_file='mysql-bin.000001', master_log_pos= 2917011, master_connect_retry=30;
    ```

    然后执行 `show slave status` 我们可以看到 Slave_IO_Running 和 Slave_SQL_Running 都是 NO，然后执行 `start slave` 开启主从复制。然后再执行 `show slave status` 我们可以看到此时 Slave_IO_Running 和 Slave_SQL_Running 都是 Yes 了。

    此时可能会遇到密码错误的问题如下：

    ![密码错误](/assets/img/mysql-ms-1.png)

    此时我们只需要在从库所在服务器登录一下主库，并且带上 `--get-server-public-key` ，例如：`mysql -uroot -h 主库IP --get-server-public-key`，然后退出主库，重新登录从库，先执行 `stop slave` 和 `reset slave` 重置 slave 节点配置，然后重新配置主库信息和开启主从复制即可;

<!-- ### Binlog

主库每提交一次事务，都会把数据变更，记录到一个二进制文件中，这个二进制文件就叫 Binlog。需注意：只有写操作才会记录至 Binlog，只读操作是不会的（如select、show语句）。

> 如何开启 Binlog？Binlog 除了主从同步是否还有其他作用？
{: .prompt-warning}


Binlog 有三种格式：

- statement 格式：binlog 记录的是实际执行的 sql 语句
- row 格式：binlog 记录的是变化前后的数据（涉及所有列），形如 update table_a set col1=value1, col2=value2 ... where col1=condition1 and col2=condition2 ...
- mixed 格式：默认选择 statement 格式，只在需要时改用 row 格式

设置 Binlog 格式：

```conf
[mysqld]
# 设置binlog格式
binlog-format="MIXED"
```
{: file='my.cnf'} -->

<!-- 
https://www.modb.pro/db/29919


如何连接？jdbc replication
https://blog.csdn.net/qq422243639/article/details/83539955

主从延迟如何解决？ -->

