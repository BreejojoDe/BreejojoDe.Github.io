---
layout: post #ensure this one stays like this
read_time: true # calculate and show read time based on number of words
show_date: true # show the date of the post
title:  Spark集群环境配置（Ubuntu）
date:   2024-2-27 # XXXX-XX-XX XX:XX:XX XXXX
description: NULL
img:  posts/20230803/githubweb.png# the path for the hero image, from the image folder (if the image is directly on the image folder, just the filename is needed)
tags: [Spark, linux, 分布式]
author: BreejojoDe
github: username/reponame/ # set this to show a github button on the post
toc: yes # leave empty or erase for no table of contents
---


## 初始化各个节点服务器
- 安装 openssh-server，修改sshd配置，允许root连接
- 查看本机ip并记录。修改 `/etc/hosts`，添加一些host（建议）

## 下载 Spark
下载后解压到对应文件夹（/opt）

### 修改Spark的配置
进入spark文件夹下的`/conf`，找到两个文件，一个是启动脚本模板`spark-env.sh.template`，一个是存节点信息的`slaves.template`或者`workers.template`。  

在conf文件夹里复制 spark-env.sh.template，去掉模板后缀，再在后面添加上：
``` sh
# nano spark-env.sh
# 学校教程里只添加了master ip
export SPARK_MASTER_HOST=192.168.44.138  #主节点IP
export SPARK_MASTER_PORT=7077  #任务提交端口
export SPARK_WORKER_CORES=2  #每个worker使用2核
export SPARK_WORKER_MEMORY=3g  #每个worker使用3g内存
export SPARK_MASTER_WEBUI_PORT=7979  #修改spark监视窗口的端口默认8080

export JAVA_HOME=/opt/java1.8.0  #当不能读取系统配置时需要自行添加配置
```

同样把slaves/workers复制到这个文件夹里，这个文件储存工作的节点信息
``` bash
# 修改slaves的内容，直接把ip或者对应host写进去就行
# 一行一个
Host0
Host1

192.168.44.140
```

## 配置ssh


## 配置zookeeper

### 数据目录

### /etc/host

### /tmp/zookeeper/data/myid

### $ZOOKEEPER_HOME/conf


## 配置kafka

### $KAFKA_HOME/conf/server.properties - broker.id

### $KAFKA_HOME/conf/server.properties - listeners
每个机器对应自己的IP

### $KAFKA_HOME/conf/server.properties - zookeeper.connect
注意格式