---
title: 使用 Jenkins + GitLab 进行代码自动部署
author: zhangzangqian
date: 2022-10-08 12:00:00 +0800
categories: [技术]
tags: [Linux, Jenkins]
math: true
mermaid: true
---

## 插件配置

### JDK、Maven

从官网下载对应的压缩包，解压至 `/data/jenkins_home/` 目录下，然后在 Jenkins 系统配置。在 **系统管理 -> 全局工具配置** 页进行配置。

截图如下

![jdk 配置](/assets/img/jk-devops-2.png)

![maven 配置](/assets/img/jk-devops-1.png)

### git

如果系统中已经安装并且可以直接调用，直接配置为 git 就可以。否则安装方式跟 jdk、maven 一致。

![git 配置](/assets/img/jk-devops-3.png)

## 项目发布

### 新建项目

在 Jenkins 主页点击左侧新建项目，填入项目名称，并选择 **构建一个自由风格的软件项目** 然后点击确定。

![新建项目](/assets/img/jk-devops-5.png)

### 源码配置

然后会跳转到项目配置页，我们找到源码管理，然后选择 Git 作为代码仓库，并配置仓库地址和认证信息。

![](/assets/img/jk-devops-6.png)

我们注意，在配置好代码仓库地址后需要配置鉴权信息，也就是 `Credentials` 下拉框，上图是已经配置好了所以直接选择，如果没有配置过则点击添加按钮配置一个，点击添加按钮后选择 Jenkins 然后会弹窗如下，类型我们选择 `Username with password`, 填入一个 Git 账号和对应密码。

### 源码编译

我们找到 **Build Steps** 项然后增加构建步骤，选择 **调用顶层 Maven 目标**，版本选择我们已经配置好的 maven 版本，目标填入打包命令 `clean package -DskipTests=true`。

### 部署服务

打包结束便是上传部署服务器然后运行，我们在 **Build Steps 项调用顶层 Maven 目标后继续增加构建步骤**，选择**执行 Shell**，内容如下，根据实际情况替换响应变量值即可。

```bash
#jenkins文件路径
JENKINS_FILE_PATH=${WORKSPACE}/target/
#父节点文件路径
TARGET_FATHER_PATH=/opt/shell/
#文件路径
TARGET_FILE_PATH=/opt/820/digital_car_factory_server
#服务名称
JAR_NAME=digital_car_factory_server.jar
#脚本名称
SH_NAME=deploy.sh
#日志输出脚本名称
TAIL_SH_NAME=tailf.sh
# 部署机器
TARGET_HOST=192.168.150.70
SERVER_PORT=8200

# 复制软件包
scp -r ${JENKINS_FILE_PATH}/${JAR_NAME} root@${TARGET_HOST}:${TARGET_FILE_PATH}
echo "copy ${JAR_NAME} success.........."

# 远程调用 deploy 脚本
ssh root@${TARGET_HOST} "cd ${TARGET_FILE_PATH} && nohup sh ${TARGET_FATHER_PATH}/${SH_NAME} restart ${TARGET_FILE_PATH} ${JAR_NAME} ${profile} ${SERVER_PORT} &"
echo "start ${JAR_NAME} success.........."

ssh root@${TARGET_HOST} "sh ${TARGET_FATHER_PATH}/${TAIL_SH_NAME} ${TARGET_FATHER_PATH} ${TARGET_FILE_PATH}"
```

通过注释我们可以看到 shell 主要实现了三件事情：

1. 发送 Jar 包到部署服务器
2. 远程调用 deploy 脚本启动服务
3. 执行日志输出脚本，在 Jenkins 控制台打印系统启动日志，用于监控服务器启动。

如此，我们的自动部署配置便完成了。剩下就是将脚本上传到部署服务器即可（只需上传一次）。


### 相关脚本

1. 服务启动脚本

    ```bash
    #! /bin/bash
    #文件路径
    FILE_PATH=$2
    #服务名称
    JAR_NAME=$3
    #启动环境
    PROFILE=$4

    PORT=$5

    PID_FILE=$FILE_PATH/application.pid

    function start(){
        cd $FILE_PATH
        rm nohup.out
        echo "remove nohup.out result:" + $?
        nohup java -Dfile.encoding=UTF-8 -jar ${FILE_PATH}/${JAR_NAME} --spring.profiles.active=${PROFILE} --server.port=${PORT}  > ${FILE_PATH}/nohup.out 2>&1 &
        if [[ $? -eq 0 ]]
        then
            echo "start agent success.........."
        else
            echo "start agent failed..........."
        fi
        exit 0
    }

    function stop(){
        if [ -e $PID_FILE ]
        then 
            kill `cat $PID_FILE`
            echo "stop agent success.........."
        else 
            ps -ef | grep ${FILE_PATH}/$JAR_NAME |grep -v deploy| awk '{print $2}' | xargs kill
            if [[ $? -eq 0 ]]
        then
        echo "stop agent success............"
        else
        echo "stop agent failed............."	
        fi
        fi
    }
    case $1 in
        start)
            start
        ;;
        stop)
            stop
        ;;
        restart)
            stop
            sleep 2
            start
        ;;
        *)
            echo "Usage sh agent_process {start|stop|restart}"
        ;;
    esac
    ```
    {: file='deploy.sh'}

2. 打印日志脚本

    ```bash
    #!/bin/bash
    KILL_TAILF_SHELL_PATH=$1
    LOG_FILE_PATH=$2
    # $1为 kill_tailf.sh 脚本的路径
    sh $KILL_TAILF_SHELL_PATH/kill_tailf.sh $2 &

    # 执行 tailf 查看日志
    echo "执行shell命令： tailf $LOG_FILE_PATH/nohup.out"
    tailf $LOG_FILE_PATH/nohup.out
    echo [tags] tailf 命令已经成功退出

    # 判断是否捕获到 Started .* second
    sum=`cat $LOG_FILE_PATH/boole_log`
    sum_jvm=`cat $LOG_FILE_PATH/boole_jvm_log`

    if [ $sum -gt 0 ];then
        echo [tags] ------------
        echo [tags] 本次部署成功
        echo [tags] ------------
    elif [ $sum_jvm -gt 0 ];then
        echo [tags] ------------
        echo [tags] 本次部署成功
        echo [tags] ------------
    else
        echo [tags] ------------
        echo [tags] 本次部署失败
        echo [tags] ------------
        exit 1
    fi
    ```
    {: file='tailf.sh'}

3. 服务部署接口监控接口，会在打印日志脚本中后台启动，监控日志关键字，如果监控到相应关键字，则终止 `tailf.sh` 脚本中的 `tailf` 进程，使日志打印脚本正常退出，否则超时也会退出。
    
    ```bash
    #!/bin/bash
    #项目日志文件 nohup.out
    LOG_FILE_PATH=$1

    des_log=$LOG_FILE_PATH/nohup.out

    echo "项目日志为：$des_log"
    second=0
    echo "" > $LOG_FILE_PATH/boole_log
    echo "" > $LOG_FILE_PATH/boole_jvm_log

    while true
    do
        sum=`cat ${des_log} |grep 'Started ScaffoldApplication in' |wc -l`
        sum_jvm=`cat ${des_log} |grep 'JVM running for' |wc -l`
        echo "[tags] sum: ${sum} second: $second sum_jvm: $sum_jvm"

        echo "${sum}" >$LOG_FILE_PATH/boole_log
        echo "${sum_jvm}" >$LOG_FILE_PATH/boole_jvm_log

        if [ ${second} -ge 60  ];then
        echo 部署等待时间过长 退出部署
        ps -ef |grep "tailf ${des_log}" |grep -v grep|awk '{print $2}' |xargs kill
        break
        fi
        
        if [ ${sum} -gt 0 ] || [ ${sum_jvm} -gt 0  ];then
        echo "[tags] sum ${sum} "
            ps -ef |grep "tailf ${des_log}" |grep -v grep|awk '{print $2}' |xargs kill
            #echo [tags] 项目启动花费 $second 秒   
            break
        fi
        
        ((second+=2))
        sleep 2
        #echo [tags] 启动时长second $second
    done
    ```
    {: file='kill_tailf.sh'}
