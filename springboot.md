# Springboot

## 数据库客户端相关配置

### 数据库ssl参数

```yaml
verifyServerCertificate=true&useSSL=true&requireSSL=true&trustCertificateKeyStorePassword=apsaradb&trustCertificateKeyStoreUrl=file:/shrdata/mysql-cert/ApsaraDB-CA-Chain.jks&allowMultiQueries=true&useUnicode=true&characterEncoding=utf-8&serverTimezone=GMT%2B8
```

### Druid相关配置与加密

```yaml
  datasource:
    type: com.alibaba.druid.pool.DruidDataSource
    driver-class-name: com.mysql.cj.jdbc.Driver
    url: jdbc:mysql://${APP_DB_URL:}/${APP_DB_SCHEMA:}?${APP_DB_PARAM:allowMultiQueries=true&useUnicode=true&characterEncoding=utf-8&useSSL=false&serverTimezone=GMT%2B8}
    username: 
    password: 
    druid:
      initialSize: 8
      minIdle: 8
      maxActive: 200
      maxWait: 60000
      timeBetweenEvictionRunsMillis: 60000
      minEvictableIdleTimeMillis: 300000
      validationQuery: SELECT 1
      testWhileIdle: true
      testOnBorrow: false
      testOnReturn: false
      poolPreparedStatements: true
      maxPoolPreparedStatementPerConnectionSize: 20
      # 配置监控统计拦截的filters，去掉后监控界面sql无法统计，'wall'用于防火墙
      filters: stat,wall,slf4j,config
      # 通过connectProperties属性来打开mergeSql功能；慢SQL记录
      connectionProperties: druid.stat.mergeSql=true;druid.stat.slowSqlMillis=5000;config.decrypt=${APP_ENABLE_DB_PASS_ENCRYPT:false};config.decrypt.key=${APP_DB_PASS_ENCRYPT_KEY:MFwwDQYJKoZIhvcNAQEBBQADSwAwSAJBAMWC7ke7VjPC35VLK6akDeZ1U4oU24HyYUEsMxMbVX889mf7w7ANjlPVCVV2GHRX6g8qKB7mm66fwmKinNriHQ8CAwEAAQ==}
      monitor:
        loginUsername: druid
        loginPassword: druid123

```

### Oracle与mysql验证sql不同

```yaml
      validationQuery: SELECT 1
      validationQuery: SELECT 1 FROM DUAL
```

### 数据库连接池优化

- 服务端内核数x2+1,来源于postgresql官方推荐.实际根据压测
