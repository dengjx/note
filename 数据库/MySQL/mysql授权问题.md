1. MySQL授权远程访问

   ```GRANT ALL PRIVILEGES ON *.* TO 'ROOT'@'%' IDENTIFIED BY 'ROOT' WITH GRANT OPTION;```

   `FLUSH PRIVILEGES;`

2. 主机Windows：授权哪个数据库作为自己的slave

   `GRANT REPLICATION slave,reload,super ON *.* TO 'root'@'192.168.2.%' IDENTIFIED BY 'root;'`

   `flush privileges;`

   `show master status;`

3. 从机Linux：授权哪个数据库作为自己的master

   `CHANGE MASTER TO MASTER_HOST='192.168.2.2',`

   `MASTER_USER='root', MASTER_PASSWORD='root',MASTER_PORT=3306,`

   `master_log_file='mysql-bin.000002',`

   `master_log_pos=667;`

4. 开启主从的远程连接权限

   `GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' IDENTIFIED BY 'root' WITH GRANT OPTION;`

   `FLUSH PRIVILEGES;`







