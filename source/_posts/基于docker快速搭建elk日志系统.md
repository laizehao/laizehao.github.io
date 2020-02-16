---
title: 基于docker快速搭建elk日志系统
date: 2019-09-15 12:25:31
tags: [教程,docker,elasticsearch,kibana,logstash]
categories: [教程,docker]
thumbnail:  https://gitee.com/minagamiyuki/picgo-gitee/raw/master/images/20200216175552.png
---

## 目的

因为公司业务要求,目前的架构是采用了`spring-cloud`微服务架构.

无论是开发,测试,还是生产环境,都需要通过日志来排查出现问题的原因.ssh上去查询日志太麻烦了,特别是一个服务有多个实例的情况下,根本不知道报错位置在哪个实例上.只能一个个上去查找.非常浪费时间.

因此需要搭建基于`elk`的日志系统,另外,采用docker搭建elk更加方便,所以采用docker来搭建elk.

## 搭建过程

1.  **镜像版本**

   ​	kibana 6.8.2

   ​	elasticsearch 6.8.2

   ​	logstash 6.8.2

2.  **脚本**

   创建elsearch,kibana ,logstash

   ```bash
   #创建子网用于服务内部通讯
   docker network create --subnet=192.119.0.0/16 multihost
   #设置文件句柄数量防止el-search启动失败
   sysctl -w vm.max_map_count=262144
   #安装elsearch
   # --net= 加入子网 multihost 并设定当前容器在子网内的ip地址为 192.119.2.51
   docker run -d --name elasticsearch  \
              --net=multihost --ip=192.119.2.51 \
              elasticsearch:6.8.2
   #安装kibana
   #--net=multihost 加入子网,地址由任意分配
   #-e ELASTICSEARCH_URL 配置el-search地址
   #-p 5601:5601 映射端口到宿主机的5601端口
   docker run -d --name mykibana \
         --net=multihost \
         -e ELASTICSEARCH_URL=http://192.119.2.51:9200 \
         -p 5601:5601 \
         kibana:6.8.2
   #安装logstash
   #指定当前路径为basedir
   basedir=`cd $(dirname $0); pwd -P`
   #-v /$basedir:/data/conf把当前路径映射到容器内的/data/conf
   docker run  -d \
               -v "$PWD":$basedir \
               -v /$basedir:/data/conf \
               --net multihost \
               logstash:6.8.2 \
               logstash -f /data/conf/logstash.conf
   ```

   **logstash.conf**,需要跟脚本放在同一目录下:

   ```
   #logstash.conf
   
   #接收数据的端口.
   input {
        tcp{
          port => 29820 #端口
          mode => "server"
          host => "0.0.0.0"
       }
   }
   
   #在将日志发送到logstash之前会将它转换成结构化的json格式.
   filter{
       json{
           source => "message"
       }
   }
   
   output {
     elasticsearch { 
           hosts => ["127.0.0.1"]
       index => "converge_log_%{+YYYY.MM.dd}"
       document_type => "verbose" 
     }
   }
   ```

