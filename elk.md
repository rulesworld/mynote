
# 基于Docker搭建ElK

[TOC]

## 1. 环境清单

- Centos 7.x

- docker- stable latest

- docker-compose latest

- Elasticsearch 7.6.2

- Logstash 7.6.2

- Kibana 7.6.2

- Filebeat 7.6.2

## 2. 环境准备

### 2.1. Linux内核上调线程数限制

```shell
#!/bin/bash

## 修改或添加vm.max_map_count = 262144，推荐2的倍数
sudo vi /etc/sysctl.conf

## 保存后立即应用
sudo sysctl -p
```

### 2.2. 关闭防火墙或放行端口

```shell
#!/bin/bash
## 停止防火墙服务
systemctl stop firewalld.service

## 禁用防火墙自启动
systemctl disable firewalld.service

## 放行80端口tcp，--permanent 立即生效
firewall-cmd --zone=public --add-port=80/tcp --permanent

## 放行80端口，指定ip或ip网段，指定端口，协议
firewall-cmd --permanent --add-rich-rule="rule family="ipv4" source address="192.168.0.0/8" port protocol="tcp" port="80" accept"

## 防火墙软重启
firewall-cmd --reload

## 查询80端口
firewall-cmd --query-port=80/tcp

##  查看已开放的端口
firewall-cmd --list-port

```

### 2.3.  禁用selinux

```shell
#!/bin/bash
## 临时关闭selinux，重启失效
setenforce 0
## 编辑selinux配置，机器重启永久生效
## 将 SELINUX=enforcing 改为 SELINUX=disabled
vi /etc/selinux/config
```

## 3. 安装docker步骤

其它Linux分支，见：[docker官网Linux安装docker教程](https://docs.docker.com/engine/install/)

### 3.1. 卸载旧版

```shell
#!/bin/bash
sudo yum remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-engine
```

### 3.2. yum添加docker源

```shell
#!/bin/bash
## 安装yum源管理工具
sudo yum install -y yum-utils

## 添加docker yum源
sudo yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo
```

### 3.3. 安装docker

```shell
#!/bin/bash
sudo yum install docker-ce docker-ce-cli containerd.io
```

### 3.4. 启动docker

```shell
#!/bin/bash
sudo systemctl start docker
```

### 3.5. 设置docker自启动

```shell
#!/bin/bash
sudo systemctl enable docker
```

## 4. 安装docker-compose

其它Linux分支，见：[docker官网Linux安装docker-compose教程](https://docs.docker.com/compose/install/) 

### 4.1. 查看最新版docker-compose

[github docker-dompose release页面](https://github.com/docker/compose/releases)
禁止使用非稳定版，如RC里程碑版

```shell
#!/bin/bash
sudo curl -L "https://github.com/docker/compose/releases/download/1.26.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
```

### 4.2. 允许二进制文件可执行

```shell
#!/bin/bash
sudo chmod +x /usr/local/bin/docker-compose
```

### 4.3. 检查环境变量

如果/usr/local/bin/不在$PATH环境变量中,建立到/usr/bin的符号连接

```shell
#!/bin/bash
echo $PATH | grep /usr/local/bin
sudo ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose
```

### 4.4. 检查docker-compose版本

```shell
#!/bin/bash
docker-compose --version
```

## 5. docker-compose安装ELK步骤

### 5.1. 单体配置

#### 5.1.1. 单体docker-compose.yml文件编写

（yaml格式强制缩进格式，注意缩进）

[docker-compose参考版本](https://docs.docker.com/compose/compose-file/compose-versioning/)

elk.yml

```yaml
## 限制python版本
version: "3.8"

services:
  ##  elasticsearch服务，单节点
  elasticsearch-service-01:
    ## 使用elastic.co官网镜像，不要使用dockerhub镜像,所有服务版本号需要保持一致
    image: docker.elastic.co/elasticsearch/elasticsearch:7.6.2
    environment:
      ## 时区
      TZ: Asia/Shanghai
      ## 集群名
      CLUSTER.NAME: elasticsearch-cluster
      ## 节点名
      NODE.NAME: elasticsearch-node-01
      ## 内存锁
      BOOTSTRAP.MEMORY_LOCK: "true"
      ## 限制内存大小
      ES_JAVA_OPT: "-xms512m -xmx512m"
      ## 发现方式，单节点
      discovery.type: single-node
      ## 获取文件的所有者权限，默认UID :1000
      TAKE_FILE_OWNERSHIP: "true"
    ## 取消文件描述符限制
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      ## 挂载data目录，   :rw   允许读写,前面是外部路径，自动创建
      - ./elasticsearch/data:/usr/share/elasticsearch/data:rw
    ## 容器名
    container_name: elasticsearch-container-01
    ## 给容器ip分配内部domain，其它容器可以直接使用该名进行网络连接
    hostname: elasticsearch-host-01
    ## 自动重启，允许
    restart: always
    ## 发布节点端口,9300是tcp连接端口，8.x后废弃，不需要暴露
    ports:
      - 9200:9200
    networks:
      - elastic-network

  ##  kibana服务，单节点
  kibana-service-01:
    image: docker.elastic.co/kibana/kibana:7.6.2
    environment:
      ## es 地址，暂不使用，使用配置文件，es容器无法传入数组变量
      # ELASTICSEARCH_URL: http://elasticsearch-container-01:9200
      # ELASTICSEARCH_HOSTS: http://elasticsearch-container-01:9200
      ## 默认1.4GB
      NODE_OPTIONS: --max-old-space-size=2048
    volumes:
      - ./kibana/kibana.yml:/usr/share/kibana/config/kibana.yml:rw
    container_name: kibana-container-01
    hostname:
      kibana-host-01
      ## 依赖于容器elasticsearch，elasticsearch完成启动后正常，才可以启动
    depends_on:
      - elasticsearch-service-01
    restart: always
    ports:
      - 5601:5601
    networks:
      - elastic-network

  ##  logstash服务，单节点
  logstash-service-01:
    image: docker.elastic.co/logstash/logstash:7.6.2
    container_name: logstash-container-01
    hostname: logstash-host-01
    restart: always
    volumes:
      ## 映射配置文件，外部会覆盖内部文件
      - ./logstash/logstash.yml:/usr/share/logstash/config/logstash.yml
      ## logstash自动默认扫描 /usr/share/logstash/config/ 下的conf配置
      - ./logstash/filebeatsinput1.conf:/usr/share/logstash/config/logstash-sample.conf
    depends_on:
      - elasticsearch-service-01
    ports:
      - 9600:9600
      - 5044:5044
    networks:
      - elastic-network

networks:
  elastic-network:
    driver: bridge
```

### 5.2. 最小集群配置(不支持HA)

最小化集群中只对Elasticsearch进行集群（所有数据持久化的部分）

#### 5.2.1. 最小集群docker-compose.yml文件编写

（yaml格式强制缩进格式，注意缩进）

elk.yml

```yaml
## compose 文件参考版本
version: "3.8"

services:
  ##  elasticsearch服务，多节点
  elasticsearch-service-01:
    ## 使用elastic.co官网镜像，不要使用dockerhub镜像,所有服务版本号需要保持一致
    image: docker.elastic.co/elasticsearch/elasticsearch:7.6.2
    environment:
      ## 时区
      TZ: Asia/Shanghai
      ## 集群名
      CLUSTER.NAME: elasticsearch-cluster
      ## 节点名
      NODE.NAME: elasticsearch-node-01
      ## 内存锁
      BOOTSTRAP.MEMORY_LOCK: "true"
      ## 限制内存大小
      ES_JAVA_OPT: "-xms512m -xmx512m"
      ## 查找其它节点
      DISCOVERY.SEED_HOSTS: elasticsearch-container-02,elasticsearch-container-03
      ## 集群所有节点
      CLUSTER.INITIAL_MASTER_NODES: elasticsearch-node-01,elasticsearch-node-02,elasticsearch-node-03
      ## 获取文件的所有者权限，默认UID :1000
      TAKE_FILE_OWNERSHIP: "true"
    ## 取消文件描述符限制
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      ## 挂载data目录，   :rw   允许读写,前面是外部路径，自动创建
      - ./elasticsearch01/data:/usr/share/elasticsearch/data:rw
    ## 容器名
    container_name: elasticsearch-container-01
    hostname: elasticsearch-container-01
    ## 自动重启，允许
    restart: always
    ## 发布节点端口,9300是tcp连接端口，8.x后废弃，不需要暴露
    ## 集群发布的外部端口不能相同
    ports:
      - 9200:9200
    ## 加入网桥
    networks:
      - elastic-network

  elasticsearch-service-02:
    image: docker.elastic.co/elasticsearch/elasticsearch:7.6.2
    environment:
      TZ: Asia/Shanghai
      CLUSTER.NAME: elasticsearch-cluster
      NODE.NAME: elasticsearch-node-02
      BOOTSTRAP.MEMORY_LOCK: "true"
      ES_JAVA_OPT: "-xms512m -xmx512m"
      ## 查找其它节点
      DISCOVERY.SEED_HOSTS: elasticsearch-container-01,elasticsearch-container-03
      CLUSTER.INITIAL_MASTER_NODES: elasticsearch-node-01,elasticsearch-node-02,elasticsearch-node-03
      TAKE_FILE_OWNERSHIP: "true"
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - ./elasticsearch02/data:/usr/share/elasticsearch/data:rw
    container_name: elasticsearch-container-02
    hostname: elasticsearch-container-02
    restart: always
    ## 集群发布的外部端口不能相同
    ports:
      - 9201:9200
    ## 加入网桥
    networks:
      - elastic-network

  elasticsearch-service-03:
    image: docker.elastic.co/elasticsearch/elasticsearch:7.6.2
    environment:
      TZ: Asia/Shanghai
      CLUSTER.NAME: elasticsearch-cluster
      NODE.NAME: elasticsearch-node-03
      BOOTSTRAP.MEMORY_LOCK: "true"
      ES_JAVA_OPT: "-Xms512m -Xmx512m"
      ## 查找其它节点
      DISCOVERY.SEED_HOSTS: elasticsearch-container-01,elasticsearch-container-02
      CLUSTER.INITIAL_MASTER_NODES: elasticsearch-node-01,elasticsearch-node-02,elasticsearch-node-03
      TAKE_FILE_OWNERSHIP: "true"
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - ./elasticsearch03/data:/usr/share/elasticsearch/data:rw
    container_name: elasticsearch-container-03
    hostname: elasticsearch-container-03
    restart: always
    ## 集群发布的外部端口不能相同
    ports:
      - 9202:9200
    ## 加入网桥
    networks:
      - elastic-network

  ##  kibana服务，单节点
  kibana-service-01:
    image: docker.elastic.co/kibana/kibana:7.6.2
    environment:
      ## es 地址，暂不使用，使用配置文件，es容器无法传入数组变量
      # ELASTICSEARCH_URL: http://elasticsearch-container-01:9200
      # ELASTICSEARCH_HOSTS: http://elasticsearch-container-01:9200
      ## 默认1.4GB
      NODE_OPTIONS: --max-old-space-size=2048
    volumes:
      - ./kibana/kibana.yml/usr/share/kibana/config/kibana.yml:rw
    container_name: kibana-container-01
    hostname:
      kibana
      ## 依赖于容器elasticsearch，elasticsearch完成启动后正常，才可以启动
    depends_on:
      - elasticsearch-service-01
    restart: always
    ports:
      - 5601:5601
    networks:
      - elastic-network

  ##  logstash服务，单节点
  logstash-service-01:
    image: docker.elastic.co/logstash/logstash:7.6.2
    environment:
      LS_JAVA_OPTS: "-Xmx256m -Xms256m"
    container_name: logstash-container-01
    hostname: logstash
    restart: always
    volumes:
      ## 映射配置文件，外部会覆盖内部文件
      - ./logstash/logstash.yml:/usr/share/logstash/config/logstash.yml
      ## 映射配置文件，logstash自动默认扫描 /usr/share/logstash/config/ 下的conf配置
      - ./logstash/filebeatsinput1.conf:/usr/share/logstash/config/logstash-sample.conf
    depends_on:
      - elasticsearch-service-01
    ports:
      - 9600:9600
      - 5044:5044
    networks:
      - elastic-network

networks:
  elastic-network:
    driver: bridge
```

### 5.3. 小规模集群配置（HA/需要Kafka）

 [x] 待更新

### 5.4. 中等规模集群配置（需要HDFS）

 [x] 待更新

### 5.5. Kibana配置

#### 5.5.1. 创建kibana目录

```shell
#!/bin/bash
mkdir kibana
touch kibana/kibana.yml
```

#### 5.5.2. 编辑kibana.yml

kibana.yml

```yaml
## 按实际地址配置，容器默认内部dns，容器名指向网桥的ip
server.host: "0.0.0.0"

## 单节点
elasticsearch.hosts: ["http://elasticsearch-container-01:9200"]

## 多节点
elasticsearch.hosts: ["http://elasticsearch-container-01:9200","http://elasticsearch-container-03:9200","http://elasticsearch-container-03:9200"]

## 配置语言，默认英文
i18n.locale: "zh-CN"
```

### 5.6. Logstash配置

#### 5.6.1. 创建logstash目录

```shell
#!/bin/bash
mkdir logstash
touch logstash/logstash.yml logstash/filebeatsinput1.conf
```

#### 5.6.2. 编辑logstash.yml filebeatsinput1.conf

logstash.yml

```yml
## 按实际地址配置，容器默认内部dns，容器名指向网桥的ip
http.host: "0.0.0.0"

## 单节点
xpack.monitoring.elasticsearch.hosts: ["http://elasticsearch-container-01:9200"]

## 多节点
xpack.monitoring.elasticsearch.hosts: ["http://elasticsearch-container-01:9200","http://elasticsearch-container-02:9200","http://elasticsearch-container-03:9200"]
```

filebeatsinput1.conf

```json
## beats方式输入（from filebeats）

input {
  beats {
    port => 5044
  }
}

## 输出（to elasticsearch ）
output {
  elasticsearch {
    ## 没有kafka，并行输入到Elasticsearch会重复
    hosts => ["http://elasticsearch-container-01:9200"]
    ## filebeats的输入名，当前使用filebeat的版本元信息作为index索引
    ## 可以使用自定义index索引
    index => "%{[@metadata][beat]}-%{[@metadata][version]}-%{+YYYY.MM.dd}"
    ## 批量发送条数，减少网络请求
    flush_size => 20000
    ## 批量发送间隔，如果10s内未达到20000，依旧发送
    idele_flush_time => 10

    #user => "elastic"
    #password => "changeme"
  }
}
```

### 5.7. 集群启动

```shell
#!/bin/bash
## 相对路径下有docker-compose.yml文件，默认读取启动，-d允许后台运行
docker-compose up -d
## 指定文件启动
docker-compose -f ./elk.yml up -d
```

#### 5.7.1. 验证集群状态

```shell
#!/bin/bash
docker ps -a
```

查看容器是否存在启动失败

#### 5.7.2. 查看docker容器日志

```shell
#!/bin/bash
## 查看单个容器日志 -f 不间断输出,--tail 200输出最后200行
docker  logs -f --tail 200 <容器ID 或 容器名>

## 查看所有编排的容器日志
docker-compose  logs -f -tail 200  <容器ID 或 容器名>
```

### 5.8. Filebeats搭建

#### 5.8.1. 安装Filebeats

在需要收集日志的服务器安装即可，注意版本号需要和ELK版本号保持一致

[Linux其它分支或32位见官网](https://www.elastic.co/guide/en/beats/filebeat/current/filebeat-installation.html)

AIX暂不支持，filebeat基于golang，5.x不支持，7.2golang支持，filebeat依赖不支持

[官网系统支持清单](https://www.elastic.co/support/matrix)

[跟踪beats项目issues](https://github.com/elastic/beats/issues/15785)

```shell
#!/bin/bash
curl -L -O https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-7.6.2-x86_64.rpm
sudo rpm -vi filebeat-7.6.2-x86_64.rpm
```

#### 5.8.2. 配置Filebeats

```shell
#!/bin/bash
vi /etc/filebeat/filebeat.yml
```

添加下面配置

```yaml
##添加或修改下面配置

 # ============================== Filebeat inputs ===============================
 # input可以配置多组，不同服务使用多个input,相同服务不同种类日志，增加paths值
 filebeat.inputs:
# 日志类型
 - type: log
   # 允许配置生效
   enabled: true
   paths:
     # 文件日志目录，支持正则，支持多路径
     - /Users/zhanghong/Downloads/git/kont-ftsbatch/log/*
     # 支持windows路径
     #- c:\programdata\elasticsearch\logs\*
   #配置一个到多个tag标记，以用于logstash区分日志，key使用service来划分系统
   tags: ["aml"]

 # ------------------------------ Logstash Output -------------------------------
  output.logstash:
   # Logstash外部ip和端口
   hosts: ["0.0.0.0:5044"]
```

#### 5.8.3. 启动Filebeats

```shell
## 启动filebeats
systemctl start filebeat
## 设置自启动
systemctl enable filebeat
```

### 5.9 集成中文分词器

IK,ICU,SmartCN中文分词器


#### 5.9.1. docker-compose.yml文件改动

（yaml格式强制缩进格式，注意缩进）

以[5.1.1. 单体docker-compose.yml文件编写](5.1.1. 单体docker-compose.yml文件编写)对比，只需要修改所有elasticsearch，覆盖原入口脚本

ek-with-ik.yml

```yaml
## 限制python版本
version: "3.8"

services:
  ##  elasticsearch服务，单节点
  elasticsearch-service-01:
    ## 使用elastic.co官网镜像，不要使用dockerhub镜像,所有服务版本号需要保持一致
    image: docker.elastic.co/elasticsearch/elasticsearch:7.6.2
    entrypoint: ["sh", "/usr/share/elasticsearch/es-plugins.sh"]
    environment:
      ## 时区
      TZ: Asia/Shanghai
      ## 集群名
      CLUSTER.NAME: elasticsearch-cluster
      ## 节点名
      NODE.NAME: elasticsearch-node-01
      ## 内存锁
      BOOTSTRAP.MEMORY_LOCK: "true"
      ## 限制内存大小
      ES_JAVA_OPT: "-xms512m -xmx512m"
      ## 发现方式，单节点
      discovery.type: single-node
      ## 获取文件的所有者权限，默认UID :1000
      TAKE_FILE_OWNERSHIP: "true"
    ## 取消文件描述符限制
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      ## 挂载入口脚本
      - ./elasticsearch/es-plugins.sh:/usr/share/elasticsearch/es-plugins.sh:rw
      ## 挂载data目录，   :rw   允许读写,前面是外部路径，自动创建
      - ./elasticsearch/data:/usr/share/elasticsearch/data:rw
    ## 容器名
    container_name: elasticsearch-container-01
    ## 给容器ip分配内部domain，其它容器可以直接使用该名进行网络连接
    hostname: elasticsearch-host-01
    ## 自动重启，允许
    restart: always
    ## 发布节点端口,9300是tcp连接端口，8.x后废弃，不需要暴露
    ports:
      - 9200:9200
    networks:
      - elastic-network

  ##  kibana服务，单节点
  kibana-service-01:
    image: docker.elastic.co/kibana/kibana:7.6.2
    environment:
      ## es 地址，暂不使用，使用配置文件，es容器无法传入数组变量
      # ELASTICSEARCH_URL: http://elasticsearch-container-01:9200
      # ELASTICSEARCH_HOSTS: http://elasticsearch-container-01:9200
      ## 默认1.4GB
      NODE_OPTIONS: --max-old-space-size=2048
    volumes:
      - ./kibana/kibana.yml:/usr/share/kibana/config/kibana.yml:rw
    container_name: kibana-container-01
    hostname:
      kibana-host-01
      ## 依赖于容器elasticsearch，elasticsearch完成启动后正常，才可以启动
    depends_on:
      - elasticsearch-service-01
    restart: always
    ports:
      - 5601:5601
    networks:
      - elastic-network

  ##  logstash服务，单节点
  logstash-service-01:
    image: docker.elastic.co/logstash/logstash:7.6.2
    container_name: logstash-container-01
    hostname: logstash-host-01
    restart: always
    volumes:
      ## 映射配置文件，外部会覆盖内部文件
      - ./logstash/logstash.yml:/usr/share/logstash/config/logstash.yml
      ## logstash自动默认扫描 /usr/share/logstash/config/ 下的conf配置
      - ./logstash/filebeatsinput1.conf:/usr/share/logstash/config/logstash-sample.conf
    depends_on:
      - elasticsearch-service-01
    ports:
      - 9600:9600
      - 5044:5044
    networks:
      - elastic-network

networks:
  elastic-network:
    driver: bridge
```

#### 5.9.2.  编写IK插件安装入口

检查IK分词器版本，和ELK版本一致，未由elasticsearch plugin官方维护

[IK分词器官网releases界面](https://github.com/medcl/elasticsearch-analysis-ik/releases)

es-plugins.sh

```shell
#!/bin/bash
plugins_dir=/usr/share/elasticsearch/plugins

## 如果挂载的已经下载，则不重新安装
if [ ! -d "$plugins_dir/plugins/analysis-icu" ] ; then
  bin/elasticsearch-plugin install analysis-icu
fi
if [ ! -d "$plugins_dir/plugins/analysis-smartcn" ] ; then
  bin/elasticsearch-plugin install analysis-smartcn
fi
if [ ! -d "$plugins_dir/plugins/analysis-iku" ] ; then
  bin/elasticsearch-plugin install --batch https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v7.6.2/elasticsearch-analysis-ik-7.6.2.zip
fi

## 执行原入口脚本
exec /usr/local/bin/docker-entrypoint.sh  elasticsearch
```

### 5.10. 集群启动

```shell
#!/bin/bash
## 相对路径下有docker-compose.yml文件，默认读取启动，-d允许后台运行
docker-compose up -d
## 指定文件启动
docker-compose -f ./ek-with-ik.yml up -d
```

#### 5.10.1. 验证集群状态

```shell
#!/bin/bash
docker ps -a
```

查看容器是否存在启动失败

#### 5.10.2. 查看docker容器日志

```shell
#!/bin/bash
## 查看单个容器日志 -f 不间断输出,--tail 200输出最后200行
docker  logs -f --tail 200 <容器ID 或 容器名>

## 查看所有编排的容器日志
docker-compose  logs -f -tail 200  <容器ID 或 容器名>
```

注：

推荐网站，docker-elk开源二次封装

`https://github.com/deviantony/docker-elk/tree/searchguard#host-setup`



### 6. docker-comose安装Kafka

Kafka依赖于zookeeper,需要先搭建zookeeper集群

#### 6.1 集群Zookeeper docker-compose编写

zookeeper作为kafka集群协调者，自身必须集群，节点数必须为2n+1，最小集群数目3

zk.yml

```yaml
## compose 文件参考版本
version: "3.8"

services:
  zookeeper-service-01:
    image: zookeeper
    restart: always
    hostname: zookeeper-host-01
    container_name: zookeeper-container-01
    ports:
      - 12181:2181
    volumes:
      - ./zookeeper-01/data:/data
      - ./zookeeper-01/datalog:/datalog
    environment:
      ZOO_MY_ID: 01
      TZ: Asia/Shanghai
      ZOO_PORT: 12181
      ZOO_SERVERS: server.01=0.0.0.0:2888:3888;2181 server.02=zookeeper-container-02:2888:3888;2181 server.03=zookeeper-container-03:2888:3888;2181
    networks:
      - zookeeper-network

  zookeeper2:
    image: zookeeper
    restart: always
    hostname: zookeeper-host-02
    container_name: zookeeper-container-02
    ports:
      - 12182:2181
    volumes:
      - ./zookeeper-02/data:/data
      - ./zookeeper-02/datalog:/datalog
    environment:
      ZOO_MY_ID: 02
      TZ: Asia/Shanghai
      ZOO_PORT: 12182
      ZOO_SERVERS: server.01=zookeeper-container-01:2888:3888;2181 server.02=0.0.0.0:2888:3888;2181 server.03=zookeeper-container-03:2888:3888;2181
    networks:
      - zookeeper-network

  zookeeper3:
    image: zookeeper
    restart: always
    hostname: zookeeper-host-03
    container_name: zookeeper-container-03
    ports:
      - 12183:2181
    volumes:
      - ./zookeeper-03/data:/data
      - ./zookeeper-03/datalog:/datalog
    environment:
      ZOO_MY_ID: 03
      TZ: Asia/Shanghai
      ZOO_PORT: 12183
      ZOO_SERVERS: server.01=zookeeper-container-01:2888:3888;2181 server.02=zookeeper-container-02:2888:3888;2181 server.03=0.0.0.0:2888:3888;2181
    networks:
      - zookeeper-network

networks:
  zookeeper-network:
    driver: bridge
```





#### 6.2 集群Kafka docker-compose编写



