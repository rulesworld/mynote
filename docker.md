# Docker

## Docker 安装

<https://docs.docker.com/engine/install>

## 清空本地镜像

```shell
#本地未使用的镜像
docker images | awk '{print $3 }' | xargs docker rmi

#编译覆盖的镜像
docker images |grep none|awk '{print $3 }' | xargs docker rmi
```

## docker常用命令

```shell
#打印容器 -a包含停止的容器
docker ps
#查看镜像
docker images
#删除容器
docker rm <name或者id>
#删除镜像
docker rmi
#查看容器所有日志
docker logs <name或者id>
#进入容器内部
docker exec -it <name或者id> /bin/sh
```

## DockerFile Java Springboot摸版

```Dockerfile
# 版本控制
# syntax=docker/dockerfile:1
# 基础镜像,ARG动态化，方便升级
ARG BASE_JVM=adoptopenjdk/openjdk8-openj9:x86_64-alpine-jdk8u275-b01_openj9-0.23.0-slim
# 一个Dockerfile多镜像编译
FROM $BASE_JVM AS builder
# 维护者
LABEL version="1.0"
LABEL description="This text illustrates"
LABEL org.opencontainers.image.authors="z909920082@outlook.com"
# kompose集成到k8s
# 详见https://github.com/kubernetes/kompose/blob/master/docs/user-guide.md
LABEL kompose.service.type=nodeport
      

# 项目名称，关联镜像名称，统一命名的工程，可以动态化引入
ARG PROJECT_NAME
# 内部用户id，root为0
ARG UID=0
#内部用户名
ARG USERNAME=root
#端口动态化
ARG PORT=8080
#JVM参数
ENV PROFILES="dev" JAR_NAME=$PROJECT_NAME JVM_OPTIONS="-XX:MetaspaceSize=256m \
                                                            -XX:MaxMetaspaceSize=256m \
                                                            -Xms1024m \
                                                            -Xmx1024m \
                                                            -Xmn512m \
                                                            -Xss512k \
                                                            -XX:SurvivorRatio=8 \
                                                            -XX:+UseConcMarkSweepGC \
                                                            -Duser.timezone=GMT+8 \
                                                            -XX:+HeapDumpOnOutOfMemoryError
                                                            -Djavax.net.ssl.trustStore=/app/truststore.jks 
                                                            -Djavax.net.ssl.trustStorePassword=changeit"
#用户/目录
RUN addgroup -g $UID -S $USERNAME && \
    adduser $USERNAME -D -G $USERNAME -u $UID -s /bin/sh -h /app && \
    mkdir -p /app/logs/$PROJECT_NAME  && \
    chown -R $USERNAME:$USERNAME /app
USER $USERNAME
# 容器内默认的数据卷，可以在容器启动时指定-v参数覆盖
VOLUME /app

# jenkins 远程构建或者单jar包构建，使用以下配置,如果使用idea 插件远程构建时，
# 请将配置中的Context folder 配置为jar包打包目录
COPY $JAR_NAME.jar  /app/


# 声明暴露端口，提示开发人员
EXPOSE ${PORT}/tcp

# 切换工作目录，类似cd 可使用多次
WORKDIR /app

# 健康检查
HEALTHCHECK --interval=5s --timeout=3s --start-period=5s --retries=3\
              CMD curl -fs http://127.0.0.1:$PORT/actuator/health || exit 1

ENTRYPOINT ["/bin/sh","-c","java -jar -Djava.security.egd=file:/dev/./urandom -Dspring.profiles.active=$PROFILES $JVM_OPTIONS  /app/$JAR_NAME.jar > /dev/null"]
# 不能使用nohup 重定向输出到nohup.out, 如果需要查看实时日志，借助docker logs container-id -f
# 因为容器就是前台程序，nohup挂起后就相当于退出了sh，也就退出了容器
# ENTRYPOINT ["/bin/sh","-c", "nohup java -jar /app/app.jar > nohup.out 2>&1 &"]

```

## docker命令编译DockerFile

```shell
#编译一个新的
docker build -f </path/to/a/Dockerfile> -t project_name/app:1.0.0 --target builder  -t private_docker_register/project_name/app:1.0.0 . 

```

## .dockerignore示例

```.dockerignore
# 目的：提升docker编译速度
# 做法：排除所有，包含需要的
# Reference: https://stackoverflow.com/questions/28097064/dockerignore-ignore-everything-except-a-file-and-the-dockerfile

# Ignore everything
**

# Allow files and directories
!build/libs/*-SNAPSHOT.jar
!Dockerfile
```

## docker-compose.yaml示例

```yaml
  service_app:
    container_name: app-$ENV
    build:
      context: $APP_DIR/app
      # 如果名字不叫Dockerfile
      dockerfile: Dockerfile-alternate
      args:
        - BASE_JVM=$BASE_JVM
        - PROJECT_NAME=app
      cache_from:
        - alpine:latest
      network: host
      target: builder
      # kcompose翻译到k8s
      # 详见https://github.com/kubernetes/kompose/blob/master/docs/user-guide.md
      labels:
        kompose.service.type: nodeport
    image: app:$APP_VERSION
    #just note,can not chage when use host network
    network_mode: host
    expose:
      - "8080"
    volumes:
      - "/etc/localtime:/etc/localtime:ro"
      - "./logs:/app/logs"
      - "./truststore.jks:/app/truststore.jks"
    environment:
      - PROFILES=$PROFILES
      - OTHERS=$OTHERS
    volumes:
      - type: bind
        source: ./truststore.jks
        target: ./truststore.jks:/app/truststore.jks
        read_only: true
      - type: volume
        source: app
        target: /app/logs 
        ## 取消文件描述符限制
    ulimits:
      memlock:
        soft: -1
        hard: -1
    restart: always
    depends_on:
      - service_other_app
    logging:
       driver: "json-file"
       options:
         max-size: "1m"
    deploy:
      replicas: 1
      update_config:
        parallelism: 2
        delay: 10s
      restart_policy:
        condition: on-failure
    # 安全配置，暂时不清楚    
    securityContext:
      privileged: true
      capabilities:
        add: ["NET_ADMIN","NET_RAW"]
    configs:
      - my_config
      - my_other_config

configs:
  my_config:
    file: ./my_config.txt
  my_other_config:
    external: true

networks:
  overlay:
```

## 空环境安装网络工具

```shell
 yum update -y && yum install iproute2 net-tools iputils-ping telnet vim -y
 apt update -y && apt install iproute2 net-tools iputils-ping telnet vim -y
```
