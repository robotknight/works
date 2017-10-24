# mysql5.6 主从复制部署


## master 主数据库 vim /etc/my.cnf
	log-bin=mysql-bin			#开启二进制日志
	server-id=1  				#建议使用ip最后一段
	#binlog-ignore-db=information_schema
	#binlog-ignore-db=performance_schema
	binlog-ignore-db=mysql      #要忽略的数据库名称
	binlog-do-db=demo 			#要同步的数据库名称
	

# 命令行操作 开放master服务mysql用户远程权限

	GRANT FILE ON *.* TO 'root'@'172.17.0.3' IDENTIFIED BY '123';
	GRANT REPLICATION SLAVE ON *.* TO 'root'@'172.17.0.3' IDENTIFIED BY '123';
	FLUSH PRIVILEGES
	show master status;


## slave 从数据库 vim /etc/my.cnf
	log-bin=mysql-bin 
	server-id=3
	read_only=1
	#binlog-ignore-db=information_schema
	#binlog-ignore-db=performance_schema
	#binlog-ignore-db=mysql
	#replicate-do-db=le_data1
	#replicate-ignore-db=mysql
	#log-slave-updates
	#slave-skip-errors=all
	#slave-net-timeout=60
	#sync_binlog=1	
	#innodb_buffer_pool_size=512M	
	#innodb_flush_log_at_trx_commit=1	
	#sql_mode=NO_ENGINE_SUBSTITUTION,STRICT_TRANS_TABLESlo#lower_case_table_names=1	
	#log_bin_trust_function_creators=1
	

# 命令行操作

	stop slave;  #关闭Slave
	change master to 
	master_host='172.17.0.2',
	master_user='root',
	master_password='123',
	master_log_file='mysql-bin.000003', master_log_pos=876;
	start slave;  #开启Slave
	flush infos with read lock; #锁住表，禁止插入数据，然后查看二进制坐标起始位置
	show slave status;
	unlock infos;#解锁表



#master mysql:
#master服务器中创建数据库demo.sql，并且导入到slave服务器的数据库中。
	CREATE TABLE `infos` (
	  `id` int(11) NOT NULL AUTO_INCREMENT,
	  `title` varchar(120) NOT NULL,
	  `code` int(11) DEFAULT NULL,
	  PRIMARY KEY (`id`)
	) ENGINE=InnoDB AUTO_INCREMENT=200001 DEFAULT CHARSET=utf8;

#使用存储过程批量插入测试数据
	create procedure  BatchInsertInfos(IN start INT,IN loop_time INT)
		BEGIN
			DECLARE Var INT;
			DECLARE ID INT; 
			SET Var =0;
	    SET ID = start;
	    WHILE Var < loop_time DO
					insert into infos(id,code,title) values (null,ID,CONCAT('title_',RAND(ID)*1000 mod 200));
			SET Var = Var + 1;
	        SET ID = ID +1;
		END WHILE;
	END;


#调用存储过程,插入20万条数据
	call BatchInsertInfos(1,200000);

#php测试数据同步问题反馈
测试方案：
同时查询master和slave 数据库（demo）下的 infos表中数据。

1. 同时查询infos表中最后数据的id

2. 同时查询infos表中当前总行数

3. 关闭slave服务器下的mysql，之后重启，查看数据是否重新同步

4. master服务器表中操作 增，删，改 操作。查看slave服务器表中数据   是否同步

