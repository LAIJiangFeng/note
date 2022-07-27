### 							    							 mysql优化总结

#### 1.逻辑架构简介

######  	1.连接层

​			最上层，主要负责客户端的连接服务，完成连接处理、授权认证、及相关安全方案，引入了线程池的概念，为通过认证安全接入的客户端提供线程。

###### 	 2.服务层

​			第二层主要完成大多核心服务功能，如缓存查询、sql优化、过程、函数等。在该层，服务器会解析查询创建相应的内部树，并优化查询顺序，判断是否能			使用索引。

###### 	3.引擎层

​			存储引擎层，真正负责数据的存储和提取，服务器通过api与存储引擎进行通信。不同的存储引擎有不同的功能，这样我们可以根据需求进行选取不同的引			擎，实现性能更佳，如MyISAM、innoDB等。

###### 	4.存储层

​			数据存储层，主要将数据存储运行于裸设备的文件系统之上，并完成与存储引擎的交互。



#### 2.mysql索引优化

######    1.mysql加载顺序

​		from -on-join-where-group by-having- select-order by- limit(由先到后)

###### 	2.索引是什么？

​		索引是mysql高效获取数据的数据结构 ，如果没有特别指明，索引都是b树结构组织索引。

###### 	3.索引的优劣势

​		优势：提高数据检索效率，降低io成本,降低排序成本，降低cpu消耗。

​		劣势：索引需要占用空间，对表进行insert、update、delete会变慢。

###### 	4.索引的创建、删除、查看

​			create [unique] index 索引名 on 表名(列名，列名 ...)

​			drop index 索引名 on 表名

​			show index from 表名 \G (\G表示换行)

​		alter方式:

​			alter table 表名 add index  索引名(列名，列名 ...)

###### 	5.那些情况适合索引

​		经常查询的字段、排序字段、分组的字段、where 查询条件的字段。

###### 	6.explain介绍及使用

​		id：越大优先级越高，可判断哪个sql先执行。

​		select_type: 查询类型有主键查询、普通查询、子查询、联合查询等。

​		table:这一行数据是哪张表的。

​		type:访问类型，由好到坏依次是：system>const>eq_ref>ref>fulltext>index_merge>unique_subquery>index_subquery_range>index>ALL,一般来说得保		证查询至少达到range级别,最好达到ref

​		possible_keys:可能用到的索引

​		key:实际用到的索引

​		key_len：索引字段的最大可能长度

​		ref:显示索引的哪一列被使用，如果可能是一个常数。

​		rows:扫描了多少行

​		Extra:额外信息

​			using Filesort (使用了文件排序，表示又重新排序了，大大降低性能、耗时变长，注意)

​			using temporay (不改就废了，注意)

​			using index(使用了索引，提高性能)

​			using where（使用where）

​			using join buffer(使用buffer)

​			impossible where （可能使用where）

​			等...

​		 使用: explain sql语句

###### 		7.索引不失效

​			1.遵循最佳左前缀。

​			2.不在索引列上做任何操作(计算、函数、or)

​			3.尽量使用覆盖索引，减少select *

​			4.mysql 在使用(!= 或者 > <)的时候无法使用索引会导致全表扫描

​			5.is null,is not null 也无法使用索引

​			6.like以通配符开头('%abc')mysql索引会变成全表扫描

​			7.字符不加单引号索引失效(注意)

​			8.少用or用他来连接时索引会失效

#### 3.慢日志查询

###### 	1.开启慢查询日志

​		查询是否开启: show variables like '%slow_query_log%';

​		开启: set global slow_query_log=1;(关闭mysql服务后慢查询日志会自动关闭)

​		设置阈值: set global long_query_time=3;修改阈值到3秒就是慢会被记录的日志里

​        找到日志文件: cd var/lib/mysql

​		查看：cat  xxx.log,可以看到大于等于3秒的sql记录

​	    mysqldumpslow使用:[mysqldumpslow基本使用 - h_s - 博客园 (cnblogs.com)](https://www.cnblogs.com/zhs0/p/10521043.html)

#### 4.show profile

###### 	1.开启show profile

​		查看:show variables like 'profiling';

​		开启:set  profiling=on;

​		诊断:show profiling cpu,block io for query 10;诊断近十条sql的cpu, io

#### 5.锁

###### 	1.表锁

​		1.加表锁: lock table 表名 read(write) ,表名2 read(write),其他；

​		2.解表锁:unlock tables;

​		3.查看表锁:show open tables (in_use 为1表示有锁)

​		4.读锁:读共享，写排他.

​		5.写锁：排他锁,其他线程查询阻塞

​		6.分析表锁定: show status like 'table%';

​			table_locks_immediate:产生表级锁定的次数

​			table_locks_waited:出现表级锁定等待的次数

###### 	2.行锁（mysql默认是行锁）

​		1.索引列如果是varchar类型查询没加''直接行锁变表锁，大罪！！！

​		2.间隙锁的危害:如果是范围查询，即使没有mysql也会在该范围加锁，会导致操作该范围的数据阻塞。	

​		3.如何锁定一行: select * from 表名  where 列名=xxx for update,这样该表的该列就锁了。

​		4.行锁分析:show status like 'innodb_row_lock%';

​			innodb_row_lock_current_waits:当前正在等待锁定的数量

​			innodb_row_lock_time:从系统锁定到现在的总时长

​			innodb_row_lock_avg:每次等待所花的平均时间

​			innodb_row_lock_time_max:等待最长的一次时间

​			innodb_row_lock_waits:从系统启动到现在总共的等待次数



​			











​		







​			

​		

​			