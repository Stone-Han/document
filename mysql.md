ubuntu下mysql简单使用
# 安装 #
	sudo service mysql start
	mysql:unrecongnized service
	sudo apt-get install mysql-server

中间会设置一下root的密码
测试是否安装：

	mysql -V
注意V是大写


# 启动服务 #
	
	sudo service mysql start
# 登录 #

	mysql -u root -psho
	//输入密码
	Welcome to the MySQL monitor.  Commands end with ; or \g.
	Your MySQL connection id is 48
	Server version: 5.5.55-0ubuntu0.14.04.1 (Ubuntu)

- 注意这里命令要以;结束
	
# 查看数据库 #
	
	show databases;
	
# 新建数据库 #

	create database student;
	
# 新建表 #
	
	use student;
	create table s(loginID varchar(20), ID varchar(18), loginDate DATETIME );
	show tables;
	
	desc s;	//查看表结构
	select * from s;  	//查看表中的内容
	
	+---------+------+---------------------+
	| loginID | ID   | loginDate           |
	+---------+------+---------------------+
	| 1       | 232  | 0000-00-00 00:00:00 |
	| 2       | 212  | 0000-00-00 00:00:00 |
	| 3       | 202  | 0000-00-00 00:00:00 |
	| 3       | 202  | 0000-00-00 00:00:00 |
	| 3       | 202  | 0000-00-00 00:00:00 |
	+---------+------+---------------------+

	
	select *  from s  where SubString(ID,2,1) in (1,3,5,7,9);//查看表中ID字段倒数第二位为奇数的单元
	select *  from s  where SubString(ID,2,1) in (0,2,4,6,8);
	select *  from s  where SubString(ID,2,1) in (0,2,4,6,8) group by loginID;//每个ID只计数一次
	select *  from s  where SubString(ID,2,1) in (1,3,5,7,9);//对上面计数
	
利用上面的语句可以通过身份证倒数第二位判断性别，然后对性别进行计数。
但是这样要对数据库进行两次查询
 
	select sum( case when SubString(ID,2,1) in (0,2,4,6,8) then 1 else 0 end) as women from s ;
	select sum( case when SubString(ID,2,1) in (0,2,4,6,8) then 1 else 0 end) as women, sum( case when SubString(ID,2,1) in (1,3,5,7,9) then 1 else 0 end) as men from s ;
	+-------+------+
	| women | men  |
	+-------+------+
	|     3 |    2 |
	+-------+------+
把sum 换count结果是5,5

一次统计出了男女人数，但是loginID可能有重复的，如何只计算一次?
加上group by loginID 变成下面

	+-------+------+
	| women | men  |
	+-------+------+
	|     0 |    1 |
	|     0 |    1 |
	|     3 |    0 |
	+-------+------+
	
		select sum( case when SubString(ID,2,1) in (0,2,4,6,8) then 1 else 0 end) as women, 
		            sum( case when SubString(ID,2,1) in (1,3,5,7,9) then 1 else 0 end) as men 
		            from (select distinct * from s) as ss;
		            
	+-------+------+
	| women | men  |
	+-------+------+
	|     1 |    2 |
	+-------+------+
	1 row in set (0.00 sec)

但是总感觉这样做有点麻烦啊。不知道有没有更简单的方法。
参考[https://zhidao.baidu.com/question/1959655383234345540.html](https://zhidao.baidu.com/question/1959655383234345540.html) 