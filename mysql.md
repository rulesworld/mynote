# mysql

## 密码更改

```sql

alter USER 'root'@'%' IDENTIFIED BY '123456';
flush privileges;
```

## 创建数据库

```sql
create schema test_db
collate utf8mb4_general_ci;
CREATE USER 'test_user'@'%' IDENTIFIED BY '123456';
GRANT ALL PRIVILEGES on test_db.* TO 'test_user'@'%' with  GRANT OPTION ;
flush privileges;
```

## 数据库重命名

```shell
list_table=$(mysql.exe -h 68.79.26.61 -uroot -ppassword -Ne "select table_name from information_schema.TABLES where TABLE_SCHEMA='ebank_firm_wkfl'")
OLDIFS=$IFS
#IFS="n"
for table in $list_table
do
        echo ---------------------------
        echo $table
        sql="rename table ebank_firm_wkfl.$table to ebank_wkfl.$table"
        echo $sql
        mysql.exe -h 68.79.26.61 -uroot -ppassword -e "$sql"
        echo
        echo
        echo
done
IFS="$OLDIFS"
```

## 数据库服务端配置

```conf
[mysqld]
user=mysql
character-set-server=utf8mb4
default_authentication_plugin=mysql_native_password
secure_file_priv=/var/lib/mysql
expire_logs_days=7
sql_mode=STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION
max_connections=1000
#lower_case_table_names=0
explicit_defaults_for_timestamp=OFF
binlog-format=ROW
skip-name-resolve

[client]
default-character-set=utf8mb4

[mysql]
default-character-set=utf8mb4
```

## mysql移除密码加密

```sql
ALTER USER 'root'@'%' IDENTIFIED WITH mysql_native_password BY 'root';
flush privileges;
```
