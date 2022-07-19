###  Filebeats搭建

#### 1.1. 安装Filebeats

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

#### 1.2. 配置Filebeats

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
   # 提供地址为68.79.28.207:5000
   hosts: ["0.0.0.0:5044"]
```

#### 1.3. 启动Filebeats

```shell
## 启动filebeats
systemctl start filebeat
## 设置自启动
systemctl enable filebeat
```

### 
