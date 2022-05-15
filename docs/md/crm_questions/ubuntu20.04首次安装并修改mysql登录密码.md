### ubuntu20.04首次安装并修改mysql登录密码  ###

---

#### 生产力环境 ####

>主机:Win11
>
>虚拟机：VMware16
>
>LinuxOS：Ubuntu20.04，Nat 模式，IP已固定
>
>mysql版本：8.0.29
>
>IDEA版本 18.03
>
>navicat版本：navicat16
>
>
>
>

​	

#### 遇到的问题  ####

IDEA连接不上虚拟机UbuntuOS中的mysql数据库,使用navicat也无法连接上。检查LinuxOS防火墙没有问题，IPV4和IPV6的22端口和3306端口都开着。连接工具中的host正确，root正确，密码”正确“，IDEA中database连接工具中host正确，root正确，密码”正确“，drive正确（版本不需要和MySQL版本对应），

然而始终连接失败，历经两个小时的错误尝试后发现是虚拟机中的Mysql未配置好。

老师演示的是5.2.7，我安装的是8.0.28.

由于无法定位到具体问题.我直接重装了mysql

~~~

sudo rm /var/lib/mysql/ -R			#删除mysql的数据文件
sudo rm /etc/mysql/ -R			#删除mysql的配置文件
# 这两步非常重要，百度上很多文章都没有，如果是改完了密码还好，没改重装依然是系统自定账号密码
sudo apt-get autoremove mysql* --purge			#完全删除以mysql开头的所有文件和配置
sudo apt update			#数据源更新
sudo apt insatall mysql-server		#安装最新版的MySQL-server，重装后是8.0.29

~~~

#### 注意事项 ####

当安装完毕后不做任何配置改动时使用管理员账户无需密码，直接mysql可直接进入mysql操作界面

```
zhao@zhao-virtual-machine:~$ sudo su
[sudo] password for zhao:
root@zhao-virtual-machine:/home/zhao#
#查看临时账户密码
root@zhao-virtual-machine:/etc/mysql# cat debian.cnf
[client]
host     = localhost
user     = debian-sys-maint
password = XLjv0xM6GmtOlwol
socket   = /var/run/mysqld/mysqld.sock
[mysql_upgrade]
host     = localhost
user     = debian-sys-maint
password = XLjv0xM6GmtOlwol
socket   = /var/run/mysqld/mysqld.sock
#使用临时密码登录，报错显示无权限
root@zhao-virtual-machine:/etc/mysql# mysql -u root -p
Enter password:
ERROR 1045 (28000): Access denied for user 'root'@'localhost' (using password: YES)

#用管理员权限的话直接跳过密码 直接进入mysql操作界面
root@zhao-virtual-machine:/etc/mysql# mysql
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 23
Server version: 8.0.29-0ubuntu0.20.04.3 (Ubuntu)

Copyright (c) 2000, 2022, Oracle and/or its affiliates.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> 


```

**问题**：无需使用密码即可登录，不安全

**原因：**默认安装的root的身份验证插件plugin是**auth_socket**方式，使用此方式不关心，也不需要密码。它只检查用户是否使用UNIX套接字进行连接，然后比较用户名。（PS：使用auth_socket，**服务器本地登录的时候根本不需要密码，而其他主机无论如何都登不上去**，除非配置文件中（mysqld.cnf 文件在目录‘/etc/mysql/mysql.conf.d/’中）设置skip-grant-tables）

```
mysql> select user,plugin from user;
+------------------+-----------------------+
| user             | plugin                |
+------------------+-----------------------+
| root             | auth_socket           |
| mysql.session    | mysql_native_password |
| mysql.sys        | mysql_native_password |
| debian-sys-maint | mysql_native_password |
+------------------+-----------------------+
————————————————
版权声明：本文为CSDN博主「勇敢打工人」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。
原文链接：https://blog.csdn.net/weixin_41891696/article/details/120115056
```

**解决：**修改root对应的plugin值为mysql_native_password，远程连接客户端支持的是mysql_native_password 这种身份验证方式；

~~~
mysql> update user set plugin= 'mysql_native_password' where user= 'root';
mysql> select user,plugin from user;
+------------------+-----------------------+
| user             | plugin                |
+------------------+-----------------------+
| root             | mysql_native_password |
| mysql.session    | mysql_native_password |
| mysql.sys        | mysql_native_password |
| debian-sys-maint | mysql_native_password |
+------------------+-----------------------+
4 rows in set (0.00 sec)

mysql> flush privileges;
Query OK, 0 rows affected (0.00 sec)


~~~



**修改本地密码时候出现：**ERROR 1819 (HY000): Your password does not satisfy the current policy requirements

~~~
ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY '123456';
~~~

**原因:**设置的密码不符合MySQL的密码策略

~~~
#查看mysql的密码安全策略
+--------------------------------------+--------+
		| Variable_name                        | Value  |
		+--------------------------------------+--------+
		| validate_password.check_user_name    | ON     |
		| validate_password.dictionary_file    |        |
		| validate_password.length             | 8      |//这个就是你要设置的密码长度
		| validate_password.mixed_case_count   | 1      |
		| validate_password.number_count       | 1      |
		| validate_password.policy             | MEDIUM |
		| validate_password.special_char_count | 1      |
		+--------------------------------------+--------+
		7 rows in set (0.01 sec)

#
关于 mysql 密码策略相关参数；
1）、validate_password_length  固定密码的总长度；
2）、validate_password_dictionary_file 指定密码验证的文件路径；
3）、validate_password_mixed_case_count  整个密码中至少要包含大/小写字母的总个数；
4）、validate_password_number_count  整个密码中至少要包含阿拉伯数字的个数；
5）、validate_password_policy 指定密码的强度验证等级，默认为 MEDIUM；
关于 validate_password_policy 的取值：
0/LOW：只验证长度；
1/MEDIUM：验证长度、数字、大小写、特殊字符；
2/STRONG：验证长度、数字、大小写、特殊字符、字典文件；
6）、validate_password_special_char_count 整个密码中至少要包含特殊字符的个数；
~~~

**解决**：

~~~
1 mysql> set global validate_password.policy=0;		#设置密码强度等级为最低：low
	Query OK, 0 rows affected (0.00 sec)

2 mysql> set global validate_password.special_char_count=0;		#不需要包含大/小写字母
	Query OK, 0 rows affected (0.00 sec)

3 mysql> set global validate_password.length=6;			#设置的密码长度要求
	Query OK, 0 rows affected (0.00 sec)

4 mysql> set global validate_password.mixed_case_count=0;			#设置密码不需要包含特殊字符
	Query OK, 0 rows affected (0.00 sec)



~~~

**设置完毕后再次查询 ***

```
mysql> SHOW VARIABLES LIKE 'validate_password%';
	+--------------------------------------+-------+
	| Variable_name                        | Value |
	+--------------------------------------+-------+
	| validate_password.check_user_name    | ON    |
	| validate_password.dictionary_file    |       |
	| validate_password.length             | 6     |
	| validate_password.mixed_case_count   | 0     |
	| validate_password.number_count       | 1     |
	| validate_password.policy             | LOW   |
	| validate_password.special_char_count | 0     |
	+--------------------------------------+-------+
	7 rows in set (0.01 sec)
```

**再次设置密码**

```
ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY '123456';
```

**问题：**ERROR 1396 (HY000): Operation ALTER USER failed for 'root'@'localhost'

**原因：**安全检查模式未关闭，user表中root 对应的authentication_string 值非空，只有在安全检查模式关闭且authentication_stringfei值为空时才可以修改密码

**解决：**

~~~
#查看安全检查模式是否关闭，如未关闭则关闭
mysql> show variables like 'sql_safe_updates';
+------------------+-------+
| Variable_name    | Value |
+------------------+-------+
| sql_safe_updates | OFF   |
+------------------+-------+
1 row in set (0.00 sec)

#查看authentication_string值
mysql> select user,authentication_string from mysql.user;
+------------------+------------------------------------------------------------------------+
| user             | authentication_string                                                  |
+------------------+------------------------------------------------------------------------+
| root             | *6BB4837EB74329105EE4568DDA7DC67ED2CA2AD9                              |
| debian-sys-maint | $A$005$NSZ|@w79GiobVV;5ARCVn5morWW4Pv75Dbd7NTgrH0YAEx5TFl6fgKL0B2 |
| mysql.infoschema | $A$005$THISISACOMBINATIONOFINVALIDSALTANDPASSWORDTHATMUSTNEVERBRBEUSED |
| mysql.session    | $A$005$THISISACOMBINATIONOFINVALIDSALTANDPASSWORDTHATMUSTNEVERBRBEUSED |
| mysql.sys        | $A$005$THISISACOMBINATIONOFINVALIDSALTANDPASSWORDTHATMUSTNEVERBRBEUSED |
+------------------+------------------------------------------------------------------------+
 
#设authentication_string为空
#-- 关闭安全模式下，才可以执行authentication_string为空，-- authentication_string空了的情况下，才可以真正修改密码

mysql> update user set authentication_string='' where user='root';
Query OK, 0 rows affected (0.00 sec)
Rows matched: 1  Changed: 0  Warnings: 0
#修改密码
mysql> alter user 'root'@'%' identified by '123456';
Query OK, 0 rows affected (0.00 sec)
#刷新权限，修改mysql 表代表修改权限，修改完毕必须刷新权限，否则不生效。
mysql> flush privileges;
Query OK, 0 rows affected (0.00 sec)

#退出MySQL，重启mysql
mysql>quit
sudo service mysql restart;


~~~

至此，ubuntu20.04首次安装并修改mysql登录密码出现的问题全部搞定

### 配置远程连接

#### 修改root用户远程连接权限 ####

~~~
#查看root用户连接权限
mysql> use mysql; 
Database changed
mysql> select user,host from user
    -> ;
+------------------+-----------+
| user             | host      |
+------------------+-----------+
| root             | localhost         |
| debian-sys-maint | localhost |
| mysql.infoschema | localhost |
| mysql.session    | localhost |
| mysql.sys        | localhost |
+------------------+-----------+
5 rows in set (0.00 sec)

#修改root 对应host为%，%代表任何IP都可连接
update mysql.user set host="%" where user="root";

#修改完毕后再次查看
mysql> select user,host from user
    -> ;
+------------------+-----------+
| user             | host      |
+------------------+-----------+
| root             | %         |
| debian-sys-maint | localhost |
| mysql.infoschema | localhost |
| mysql.session    | localhost |
| mysql.sys        | localhost |
+------------------+-----------+
5 rows in set (0.00 sec)
~~~

#### 取消默认绑定的 mysql 登录地址 ####

```
root@zhao-virtual-machine:/home/zhao# cd /etc/mysql/mysql.conf.d
root@zhao-virtual-machine:/etc/mysql/mysql.conf.d# vim mysqld.cnf

```

将bind-address = 127.0.0.1和mysqlX-bind-address = 127.0.0.1改为 bind-address = 0.0.0.0和mysqlX-bind-address = 0.0.0.0![image-20220514223159621](https://bolgpicture.oss-cn-beijing.aliyuncs.com/imag/image-20220514223159621.png)

**ps: **好多博主上的博客我看都是说MySQL的配置文件在**/etc/mysql/my.conf** 中，我却没有找到，最后是在**/etc/mysql/mysql.conf.d/mysqld.cnf**

找到的。

至此所有该配置的都配置完了

#### 连接测试 ####

**navicat**

![image-20220514224404286](https://bolgpicture.oss-cn-beijing.aliyuncs.com/imag/image-20220514224404286.png)

**IDEA**

![image-20220514224609038](https://bolgpicture.oss-cn-beijing.aliyuncs.com/imag/image-20220514224609038.png)



**ps:**可能遇到的其他问题

MySql 8.0.11 换了新的身份验证插件（caching_sha2_password）, 原来的身份验证插件为（mysql_native_password）。而客户端工具Navicat Premium12 中找不到新的身份验证插件（caching_sha2_password），对此，我们将mysql用户使用的  登录密码加密规则  还原成  mysql_native_password，即可登陆成功。

这个时候用[navicat](https://so.csdn.net/so/search?q=navicat&spm=1001.2101.3001.7020) 连接会报错2059 Authentication plugin 'caching_sha2_password' cannot be loaded

**ps:**通过开发虚拟机中LinuxOS的3306端口，主机可以使用远程连接工具如Navicat简单配置相应host,user,password进行远程连接，但是此做法在云服务其上极其不安全，因为当LinuxOS开放除22端口外的其他端口时，任何用户都可以不经过身份验证（使用上述方法配置时）就可以连接自己服务器上的数据库，**极其容易**遭到不法分子的恶意访问。**解决办法：**①在mysql配置文件中设定可以连接本数据库的特定IP。 ②关闭3306端口使用SSH连接（待解决！）。