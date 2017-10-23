# mysql5.6 主从复制部署


## master 主数据库 vim /etc/my.cnf
	log-bin=mysql-bin
	server-id=2
	binlog-ignore-db=information_schema
	binlog-ignore-db=performance_schema
	binlog-ignore-db=mysql
	binlog-do-db=le_data1 
	
```
# 命令行操作
GRANT FILE ON *.* TO 'root'@'172.17.0.3' IDENTIFIED BY '123';
GRANT REPLICATION SLAVE ON *.* TO 'root'@'172.17.0.3' IDENTIFIED BY '123';
FLUSH PRIVILEGES
show master status;
```

## slave 从数据库 vim /etc/my.cnf
	log-bin=mysql-bin
	server-id=3
	binlog-ignore-db=information_schema
	binlog-ignore-db=performance_schema
	binlog-ignore-db=mysql
	replicate-do-db=le_data1
	replicate-ignore-db=mysql
	log-slave-updates
	slave-skip-errors=all
	slave-net-timeout=60
	
```
# 命令行操作
stop slave;  #关闭Slave
change master to master_host='172.17.0.2',master_user='root',master_password='123',master_log_file='mysql-bin.000003', master_log_pos=876;
start slave;  #开启Slave
show slave status;
···