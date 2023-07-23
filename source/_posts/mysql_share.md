---
title: mysql数据库配置
date: 2023/07/23 10:00:00
---
# Mysql数据库配置指南
## 新建数据库
   - 编辑配置文件
   ```shell
   # mysql-server版本
   mysql  Ver 8.0.33-0ubuntu0.22.04.2 for Linux on x86_64 ((Ubuntu))
   
   # 配置文件地址
   /etc/mysql/mysql.conf.d/mysqld.cnf
   
   #修改端口号和绑定地址支持外部ip连接
   port=3378
   bind-address = 0.0.0.0
   
   #重启服务
   systemctl restart mysql
   ```
   - 账号配置
   ```sql
    -- 设置初始root密码
    ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY 'set password here';

    -- 配置root可以连接的外部ip
    use mysql;
    update user set host='your ip address' where user='root';
    flush privileges;
   ```

