# mysql5.6 主从复制部署


## master 主数据库 vim /etc/my.cnf
	log-bin=mysql-bin			#开启二进制日志
	server-id=1  				#建议使用ip最后一段
	#binlog-ignore-db=information_schema
	#binlog-ignore-db=performance_schema
	binlog-ignore-db=mysql      #要忽略的数据库名称
	binlog-do-db=demo 			#要同步的数据库名称
	#expire_logs_days = 10      #日志到期时间. 
    # 日志格式，建议mixed  
    # statement 保存SQL语句  
    # row 保存影响记录数据  
    # mixed 前面两种的结合  
    #binlog_format = mixed  
	
	

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


________________________________________________________

# [二进制日志操作]>>>:
<
二进制日志记录 MySQL 数据库中所有与更新相关的操作，即二进制日志记录了所有的 DDL（数据定义语言）语句和 DML（数据操纵语言）语句，但是不包括数据查询语句。常用于恢复数据库和主从复制。

###查看 log_bin 状态:

	mysql> SHOW VARIABLES LIKE 'log_bin%';
	+---------------------------------+-------+
	| Variable_name                   | Value |
	+---------------------------------+-------+
	| log_bin                         | OFF   |
	| log_bin_basename                |       |
	| log_bin_index                   |       |
	| log_bin_trust_function_creators | OFF   |
	| log_bin_use_v1_row_events       | OFF   |
	+---------------------------------+-------+

###启用二进制日志功能
编辑 my.cnf ，在 [mysqld] 下添加

	# Binary Logging.
	log-bin="filename-bin"
保存 my.cnf 更改，重启 MySQL 服务

#####其他相关配置：
``
max_binlog_size={4096 .. 1073741824} ;
``

设定二进制日志文件上限，单位为字节，最小值为4K，最大值为1G，默认为1G。某事务所产生的日志信息只能写入一个二进制日志文件，因此，实际上的二进制日志文件可能大于这个指定的上限。作用范围为全局级别，可用于配置文件，属动态变量。

###查看日志文件

在data目录下有一个mysql-bin.index便是索引文件，以mysql-bin开头并以数字结尾的文件为二进制日志文件。

#####[查看所有的二进制文件]：

	mysql> show binary logs;
	+------------------+-----------+
	| Log_name         | File_size |
	+------------------+-----------+
	| mysql-bin.000001 |    276665 |
	+------------------+-----------+
	1 row in set (0.03 sec)
#####[查看当前正在使用的二进制文件]：

	mysql> show master status;
	+------------------+----------+--------------+------------------+
	| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB |
	+------------------+----------+--------------+------------------+
	| mysql-bin.000003 |      107 |              |                  |
	+------------------+----------+--------------+------------------+
	1 row in set (0.00 sec)

#####[二进制日志滚动]
当 MySQL 服务进程启动、当前二进制日志文件的大小已经超过上限时、执行 FLUSH LOG 时，MySQL 会创建一个新的二进制日志文件。新的编号大1的日志用于记录最新的日志，而原日志名字不会被改变。

手动滚动命令：``flush logs``;

###查看日志详细

[查看binlog日志有几种方式]：

 1 使用``show binlog events``方式可以获取当前以及指定binlog的日志

 2 使用``mysqlbinlog``命令行。

#####  1: show binlog events方式:

		mysql> show binlog events; #默认会返回mysql-bin.000001的日志

- ##### 查看指定binlog文件的内容(``show binlog events in 'binname.xxxxx'``)

		mysql> show binlog events in 'mysql-bin.000002';

- ##### 获取指定位置binlog的内容(show binlog events from xxx)
		mysql> show binlog events in 'mysql-bin.000002'  from 107;

#####  2: mysqlbinlog命令行

mysqlbinlog在mysql/bin目录


-	查看binlog日志
-	
	[root@localhost bin]# ./mysqlbinlog ../data/mysql-bin.000003

-	按时间查看二进制日志
-    
    mysqlbinlog ../data/mysql-bin.000003 --start-datetime="2015-3-11 17:00:00"
	mysqlbinlog ../data/mysql-bin.000003 --stop-datetime="2015-3-12 17:30:00"
	mysqlbinlog ../data/mysql-bin.000003 --start-datetime="2015-3-11 17:00:00" --stop-datetime="2015-3-12 17:30:00"

-	按字节数查看二进制日志
-	
	mysqlbinlog ../data/mysql-bin.000003 --start-position=20
	mysqlbinlog ../data/mysql-bin.000003 --stop-position=200
	mysqlbinlog ../data/mysql-bin.000003 --start-position=20 --stop-position=200

-	过滤insert、update操作
-	
	mysqlbinlog ../data/mysql-bin.000003 | grep insert



# II 查看binlog日志并输出:

参考方案: [http://blog.csdn.net/leshami/article/details/41962243](http://blog.csdn.net/leshami/article/details/41962243 "mysqlbinlog提取日志")

	c、提取指定position位置的binlog日志并输出到压缩文件  
	# mysqlbinlog --start-position="120" --stop-position="332" /opt/data/APP01bin.000001 |gzip >extra_01.sql.gz  
	
	d、提取指定position位置的binlog日志导入数据库  
	# mysqlbinlog --start-position="120" --stop-position="332" /opt/data/APP01bin.000001 | mysql -uroot -p  
	
	e、提取指定开始时间的binlog并输出到日志文件  
	# mysqlbinlog --start-datetime="2014-12-15 20:15:23" /opt/data/APP01bin.000002 --result-file=extra02.sql  
	
	f、提取指定位置的多个binlog日志文件  
	# mysqlbinlog --start-position="120" --stop-position="332" /opt/data/APP01bin.000001 /opt/data/APP01bin.000002|more  
	
	g、提取指定数据库binlog并转换字符集到UTF8  
	# mysqlbinlog --database=test --set-charset=utf8 /opt/data/APP01bin.000001 /opt/data/APP01bin.000002 >test.sql  
	
	h、远程提取日志，指定结束时间   
	# mysqlbinlog -urobin -p -h192.168.1.116 -P3306 --stop-datetime="2014-12-15 20:30:23" --read-from-remote-server mysql-bin.000033 |more  
	
	i、远程提取使用row格式的binlog日志并输出到本地文件  
	# mysqlbinlog -urobin -p -P3606 -h192.168.1.177 --read-from-remote-server -vv inst3606bin.000005 >row.sql  


###expire_logs_days 参数
在 my.cnf 中配置 expire_logs_days 参数指定二进制日志的有效天数，MySQL 会自动删除过期的二进制日志。expire_logs_days 设置在服务器启动或者 MySQL 切换二进制日志时生效，因此，如果二进制日志没有增长和切换，服务器不会清除老条目。 

	#设置二进制日志的有效天数
	expire_logs_days = 5

###清除二进制日志
-	清除所有日志（不存在主从复制关系）
		
		mysql> RESET MASTER;

-	清除指定日志之前的所有日志

		mysql> PURGE MASTER LOGS TO 'mysql-bin.000003';

-	清除某一时间点前的所有日志

		mysql> PURGE MASTER LOGS BEFORE '2015-01-01 00:00:00';

-	清除 n 天前的所有日志

		mysql> PURGE MASTER LOGS BEFORE CURRENT_DATE - INTERVAL 10 DAY;


#

> 警告:

> 由于二进制日志的重要性,请仅在确定不再需要将要被删除的二进制文件，或者在已经对二进制日志文件进行归档备份，或者已经进行数据库备份的情况下，才进行删除操作，且不要使用 rm 命令删除。


##日志分析工具:

-	mysqldumpslowmysql：官方提供的慢查询日志分析工具

-	mysqlsla： hackmysql.com 推出的一款日志分析工具(该网站还维护了 mysqlreport，mysqlidxchk 等比较实用的mysql 工具)。 整体来说，功能非常强大。输出的数据报表非常有利于分析慢查询的原因，包括执行频率、数据量、查询消耗等。
	
-	myprofi：纯 php 写的一个开源分析工具.项目在 sourceforge 上。功能上，列出了总的慢查询次数和类型、去重后的 sql 语句、执行次数及其占总的 slow log 数量的百分比。从整体输出样式来看，比 mysql-log-filter 还要简洁，省去了很多不必要的内容。对于只想看 sql 语句及执行次数的用户来说，比较推荐。

-	mysql-log-filter：google code 上找到的一个分析工具，提供了 python 和 php 两种可执行的脚本。 特色功能除了统计信息外，还针对输出内容做了排版和格式化，保证整体输出的简洁。喜欢简洁报表的朋友，推荐使用一下。



