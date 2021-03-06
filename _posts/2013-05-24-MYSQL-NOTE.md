---
layout : post
category : lessons
tags : [开始]
title: MYSQL NOTE
---


主服务器(Master)：192.168.184.10
从服务器(Slave)：192.168.184.11
确认两台服务器可以互访，
可从Slave服务器远程登录Master服务器的MYSQL：mysql -h 192.168.184.10 -u root -p


1、安装、初始化MYSQL数据库

2、在主服务器上 创建同步用户，并授权
     mysql>grant all on *.* to test@192.168.184.11 identified by '123';
     mysql>flush privileges;
在Slave服务器上测试：mysql -h 192.168.184.10 -u test -p123

3、修改Master配置文件
vim /etc/my.cnf
     server-id=10                             #主、从服务器的server-id必须不同，可使用IP地址最后一位作为server-id以区分
     port=3306                                 #设置端口号
     log-bin=mysql-bin                     #启用MYSQL二进制日志文件，命名格式为'mysql-bin.*'
     binlog-do-db=数据库名                  #选择需要同步的数据库，存在多个数据库可用‘,’(逗号)隔开；不写则代表同步所有数据库
     binlog-ignore-db=数据库名              #选择不需要同步的数据库，存在多个数据库可用‘,’(逗号)隔开；不写则代表同步所有数据库
保存退出，并重启MYSQL：service mysqld restart

4、修改Slave配置文件
vim /etc/my.cnf
     server-id=11
     master-host=192.168.184.10                    #指定Master服务器地址
     master-user=test                                       #指定数据同步使用的用户
     master-password=123                              #同步用户的密码
保存退出，并重启MYSQL：service mysqld restart

5、从Master获取log-bin和Position值
     在Master上登录mysql，在mysql命令行下：
     mysql>show master status \G;

"
            File: mysql-bin.000004
        Position: 106
    Binlog_Do_DB:
Binlog_Ignore_DB:
1 row in set (0.00 sec)

"     可见，log-bin文件为：mysql-bin.000004；Position值为：106

6、在Slave上写入log-bin及Position值
     在Slave上登录mysql，在mysql命令行下：
     mysql>slave stop;                                        #停止当前同步
     mysql>change master to
     ->         master_log_file='mysql-vin.000004',              #Master上的当前log-bin文件
     ->         master_log_pos=106;                                    #Master上的当前Position值
     mysql>slave start;                                        #启动同步
     mysql>show slave status \G;                        #查看同步是否成功

"
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 192.168.184.10
                  Master_User: test
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000004
          Read_Master_Log_Pos: 106
               Relay_Log_File: localhost-relay-bin.000002
                Relay_Log_Pos: 251
        Relay_Master_Log_File: mysql-bin.000004
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
              Replicate_Do_DB:
          Replicate_Ignore_DB:
           Replicate_Do_Table:
       Replicate_Ignore_Table:
      Replicate_Wild_Do_Table:
  Replicate_Wild_Ignore_Table:
                   Last_Errno: 0
                   Last_Error:
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 106
              Relay_Log_Space: 410
              Until_Condition: None
               Until_Log_File:
                Until_Log_Pos: 0
           Master_SSL_Allowed: No
           Master_SSL_CA_File:
           Master_SSL_CA_Path:
              Master_SSL_Cert:
            Master_SSL_Cipher:
               Master_SSL_Key:
        Seconds_Behind_Master: 0
Master_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 0
                Last_IO_Error:
               Last_SQL_Errno: 0
               Last_SQL_Error:
1 row in set (0.00 sec)


可见，第11,12行的值为Yes，则表示同步成功；若任何一个为No，则同步未成功。

重启Slave的MYSQL

7、同步测试
在Master上添加新的database及table：

mysql> create database lotus;
Query OK, 1 row affected (0.00 sec)

mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| lotus              |
| mysql              |
| test               |
+--------------------+
4 rows in set (0.00 sec)

mysql> use lotus;
Database changed
mysql> create table tb1 (id int(4),name char(20),degree double(16,2));
Query OK, 0 rows affected (0.00 sec)

mysql> show tables;
+-----------------+
| Tables_in_lotus |
+-----------------+
| tb1             |
+-----------------+
1 row in set (0.00 sec)

mysql> insert into tb1 values(1,'Tom',85),(2,'Jim',90.23);
Query OK, 2 rows affected (0.00 sec)
Records: 2  Duplicates: 0  Warnings: 0

mysql> show tables;
+-----------------+
| Tables_in_lotus |
+-----------------+
| tb1             |
+-----------------+
1 row in set (0.00 sec)

mysql> select * from tb1;
+------+------+--------+
| id   | name | degree |
+------+------+--------+
|    1 | Tom  |  85.00 |
|    2 | Jim  |  90.23 |
+------+------+--------+
2 rows in set (0.00 sec)

mysql> exit


在Slave上，查看数据是否同步：

mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| lotus              |
| mysql              |
| test               |
+--------------------+
4 rows in set (0.00 sec)

mysql> use lotus;
Database changed
mysql> show tables;
+-----------------+
| Tables_in_lotus |
+-----------------+
| tb1             |
+-----------------+
1 row in set (0.00 sec)

mysql> select * from tb1;
+------+------+--------+
| id   | name | degree |
+------+------+--------+
|    1 | Tom  |  85.00 |
|    2 | Jim  |  90.23 |
+------+------+--------+
2 rows in set (0.00 sec)

可见，数据已同步到Slave中。同步完成。
