## MySQL 使用过程问题小结-4
### 出错解决方案
#### 科技网87 mysql 重启使用问题
- 登录 mysql 出错 `mysql –u username –p password`：不能通过 sock 连接 mysql `mysqldump: Got error: 2002: Can't connect to local MySQL server through socket '/var/lib/mysql/mysql.sock' (111) when trying to connect`
- 解决步骤：
    1. __查看 mysql 配置文件__ `more /etc/my.cnf` 确认 mysql.sock 文件路径的确在 /var/lib/mysql/：的确存在 `socket=/var/lib/mysql/mysql.sock`
    2. __全盘查找是否存在合适的 mysql.sock 文件__ 直接 copy 到 /var/lib/mysql/ 下试试 `find / -name   mysql.sock`：没有找到
    3. 有帖子说 mysql.sock 是 mysql 启动时自动生成的文件，__检查运行进程__ `ps -ef |grep mysqld`：未启动
    4. __启动mysql服务__ `service mysqld start` ：出错，启动失败，提示查询 `systemctl status mysqld.service` 和 `journalctl -xe` 显示细节
    5. 运行 `systemctl status mysqld.service` 和 `journalctl -xe` ：显示启动失败，但没有太多帮助
    6. __查看配置文件 mysql 运行日志位置__ `more /etc/my.cnf`：显示 `log-error=/var/log/mysqld.log`
    7. __查看mysql运行日志__ `tail -f /var/log/mysqld.log`：显示 __数据库表索引树损坏，导致无法启动 mysql__ ，它建议需要转储+删除+重新导入表以修复损坏。
    8. __强制启动 mysql__ `service mysqld start`：
        + 修改mysql配置文件`vim /etc/my.cnf`，增加 innodb_force_recovery 参数设置，默认为0，从1开始增加尝试强制启动，直至为3时，`innodb_force_recovery=3`：mysql服务终于启动成功
    9. mysql 数据恢复：导出库表结构及数据等备份，删库重建后，修改 my.cnf `#innodb_force_recovery=3`
- 参考资料
    + [解决 Can't connect to local MySQL server through socket '/var/lib/mysql/mysql.sock' (2）的方法](https://blog.csdn.net/HeatDeath/article/details/79065872)
    +   [systemctl start mysqld 失败问题](https://youguanxinqing.xyz/archives/86/) 这个人也遇到了类似的问题，不过它的核心原因是 mysql.sock.lock 这个文件的权限非 mysql:mysql
    + [官方文档：mysql强制启动恢复数据](http://dev.mysql.com/doc/refman/5.7/en/forcing-innodb-recovery.html)
    + [表损坏引起的mysql无法启动问题](https://blog.csdn.net/fairplay_li/article/details/52191285)
    + [利用innodb_force_recovery修复MySQL数据页损坏](https://www.cnblogs.com/lnlvinso/p/4596531.html)
    + [记录-Mysql无法启动及数据恢复](https://jerrychan807.github.io/2019/07/30/%E8%AE%B0%E5%BD%95-Mysql%E6%97%A0%E6%B3%95%E5%90%AF%E5%8A%A8%E5%8F%8A%E6%95%B0%E6%8D%AE%E6%81%A2%E5%A4%8D/)
    + [MySQL数据库INNODB表损坏修复处理过程分享](https://www.teakki.com/p/57e22dcba16367940da63d04)
    + [MySQL损坏page问题分析](https://yq.aliyun.com/articles/700151)
    + [MYSQL数据表损坏的原因分析和修复方法](http://blog.itpub.net/23490154/viewspace-1063250/)
    + [mysql数据库索引损坏及修复经验分享](https://www.imooc.com/article/47528)
    + [InnoDB数据损坏修复](https://blog.csdn.net/qq_35440678/article/details/60321689)

InnoDB是带有事务的存储引擎，并且其内部机制会自动修复大部分数据损坏错误，它会在服务器启动时进行修复。
不过，有时候数据损坏得很严重并且InnoDB无法在没有用户交互的情况下完成修复，在这种情况下，有--innodb_force_recovery启动选项。
该选项可以设置0~6(0 不强制修复 1是最低级别 6最高级别)。
如果发生损坏，可以从1开始尝试修复，直到可以启动服务器并且可以访问有问题的表为止.
启动后使用select into outfile将表转储到文件中，然后使用drop和create命令重新创建表，最后用--innodb_force_recovery=0重新启动服务器，然后加载文件数据。
当需要在--innodb_force_recovery选项是正数的情况下修复数据库时，错误日志通常会有明确的提示信息。

```
2020-06-14T14:12:50.454031Z 0 [ERROR] InnoDB: Corruption of an index tree: table `xxxx`.`xxxxx` index `PRIMARY`, father ptr page no 25393, child page no 25394
【nnoDB：索引树损坏：表xxxx`.xxxxx`索引`PRIMARY`，父ptr页号25393，子页号25394】
PHYSICAL RECORD: n_fields 7; compact format; info bits 0
 0: len 4; hex 82509ff5; asc  P  ;;
 1: len 6; hex 0000000c886d; asc      m;;
 2: len 7; hex a30000801217d2; asc        ;;
 3: len 11; hex 3131322e36302e312e3639; asc 112.60.1.69;;
 4: len 4; hex 80000002; asc     ;;
 5: len 4; hex 80000001; asc     ;;
 6: len 5; hex 99a60eaa00; asc      ;;

2020-06-14T14:12:50.454606Z 0 [ERROR] [FATAL] InnoDB: You should dump + drop + reimport the table to fix the corruption. If the crash happens at database startup. Please refer to http://dev.mysql.com/doc/refman/5.7/en/forcing-innodb-recovery.html for information about forcing recovery. Then dump + drop + reimport.
【InnoDB：您应该转储+删除+重新导入表以修复损坏。 如果崩溃发生在数据库启动时。 有关强制恢复的信息，请参考http://dev.mysql.com/doc/refman/5.7/en/forcing-innodb-recovery.html。 然后转储+删除+重新导入。】
```

#### INSERT插入SELECT查找表结果报错
- 出错信息：`Column count doesn't match value count at row 1` 表示存储的数据与数据库表的字段类型定义不相匹配
- 解决办法：检查段类型是否正确, 是否越界, 有无把一种类型的数据存储到另一种数据类型中；看看dao层访问数据库的sql语句是否书写错误，赋值的参数是否与字段类型一致
- 小结：__注意插入的时候 insert table 后面的 table 字段一定要一一对应__
- 参考资料：[报错：Column count doesn't match value count at row 1](https://www.cnblogs.com/yizhiamumu/p/9044523.html)

#### 插入主键冲突
- 问题描述：组合主键中，已有两者均填好的数据，但同时存在只有域名的种子数据，当查询得到结果，可以插入的时候，本应直接更新该空缺IP的域名种子行，但是插入时却发生主键冲突，__原因是已有域名和IP的结果在表中__
- 解决方案：
    1. 临时方案：直接先把表中这种重复的种子数据删掉，没有作为种子，自然就不会有重复冲突了；[Mysql删除重复记录，保留id最小的一条](https://www.cnblogs.com/youxin/p/6380234.html) `SELECT `name`,count(`name`) as count FROM `table` GROUP BY `name` HAVING count(`name`) >1 ORDER BY count DESC;`
    2. 初级方案：修改代码，在代码中加入判断 try catch 的逻辑——比较麻烦
    3. 中级方案：在其他能够插入种子序列的代码中，添加判断逻辑，如果确定是表中没有的域名再进行添加，不能直接只通过 INSERT IGNORE INTO 来判断
    4. 高级方案：把种子表和结果表完全放在两个表中，结果的插入绝对不会引起组合主键的冲突。

### 查询命令样例
#### 数据导出导入重建表恢复
- [mysql语句-select...into outfile](https://blog.csdn.net/JesseYoung/article/details/41346861) ：SELECT INTO…OUTFILE语句把表数据导出到一个文本文件中，并用LOAD DATA …INFILE语句恢复数据。但是这种方法只能导出或导入数据的内容，不包括表的结构，如果表的结构文件损坏，则必须先恢复原来的表的结构。也可以将查询结果保存在变量中。
- [MySQL 导出数据](https://www.runoob.com/mysql/mysql-database-export.html)
- 导出文件出了点小问题：ERROR 1290 (HY000): The MySQL server is running with the --secure-file-priv option so it cannot execute this statement 
    + 解决方案：[mysql5.7导出数据提示--secure-file-priv选项问题的解决方法](https://blog.csdn.net/fdipzone/article/details/78634992) 查看官方文档，secure_file_priv参数用于限制LOAD DATA, SELECT …OUTFILE, LOAD_FILE()传到哪个指定目录，__如果不修改此参数，则 导出路径只能在 /var/lib/mysql-files/ 文件夹下__
    + secure_file_priv 为 NULL 时，表示限制mysqld不允许导入或导出。
    + secure_file_priv 为 /tmp 时，表示限制mysqld只能在/tmp目录中执行导入导出，其他目录不能执行。
    + secure_file_priv 没有值时，表示不限制mysqld在任意目录的导入导出。
    + 查看 secure_file_priv 的值， mysql> `show global variables like '%secure_file_priv%';`
- [查看库表创建语句](https://blog.csdn.net/yh_zeng2/article/details/74075367)：`SHOW CREATE TABLE TEST;`
- [查看当前所在库的事件](https://blog.csdn.net/JesseYoung/article/details/35257527) ：`show events;`
- [创建事件](https://www.yiibai.com/mysql/working-mysql-scheduled-event.html)：要创建和计划新事件，请使用CREATE EVENT语句
- [MySQL导出数据库、数据库表结构、存储过程及函数](https://blog.csdn.net/lierwang2017/article/details/78744468)
    + 执行命令： mysqldump -hhostname -uusername -ppassword -ntd -R databasename > prorandfunc.sql
    + -d 结构(--no-data:不导出任何数据，只导出数据库表结构)
    + -t 数据(--no-create-info:只导出数据，而不添加CREATE TABLE 语句)
    + -n (--no-create-db:只导出数据，而不添加CREATE DATABASE 语句）
    + -R (--routines:导出存储过程以及自定义函数)
    + -E (--events:导出事件)
    + --triggers (默认导出触发器，使用--skip-triggers屏蔽导出)
    + -B (--databases:导出数据库列表，单个库时可省略）
    + --tables 表列表（单个表时可省略）
    + ①同时导出结构以及数据时可同时省略-d和-t
    + ②同时 不 导出结构和数据可使用-ntd
    + ③只导出存储过程和函数可使用-R -ntd
    + ④导出所有(结构&数据&存储过程&函数&事件&触发器)使用-R -E(相当于①，省略了-d -t;触发器默认导出)
    + ⑤只导出结构&函数&事件&触发器使用 -R -E -d
    + mysqldump -uroot -p111111 -R -E -d dns_passive_hcy > dns_passive_hcy.sql
- [如何处理数据迁移过程中自增列后报错：主键重复](https://support.huaweicloud.com/ddm_faq/ddm_04_0035.html)
- [MySQL 导入数据-菜鸟教程](https://www.runoob.com/mysql/mysql-database-import.html)
- [mysql自增长数据怎么导入导出](https://bbs.csdn.net/topics/300069636)  ：mysqldump出结果，再复制到一些有正则表达式的IDE里，用正则表达式删掉ID字段跟ID的值
- [使用mysqlimport导入包含主键自增长属性的表](https://blog.51cto.com/mckobe23/971185)  

```sql
-- 导出数据
/usr/bin/mysqldump -u [用户名] -p [密码] [数据库名] [表名] | gzip > ./[表名].sql.gz -- 用 mysqldump 导出的是带有库表格式的数据
mysqldump -u root -p111111 -d dns_passive_hcy > dns_passive_hcy.sql # 仅导出库表结构
SELECT * FROM [表名] INTO OUTFILE '/var/lib/mysql-files/[文件名].csv' FIELDS TERMINATED BY '\t' ENCLOSED BY '"' LINES TERMINATED BY '\n'; -- 这个导出只有纯数据的结果，既然是索引出了问题，导出来的数据需要把自增索引删掉，处理之后再导入到库表里。

-- 查看导出文件可用目录
mysql> show global variables like '%secure_file_priv%';
+------------------+-----------------------+
| Variable_name    | Value                 |
+------------------+-----------------------+
| secure_file_priv | /var/lib/mysql-files/ |
+------------------+-----------------------+
1 row in set (0.00 sec)

SELECT * FROM runoob_tbl 
INTO OUTFILE '/tmp/runoob.txt';

SELECT ... INTO OUTFILE 语句有以下属性:
LOAD DATA INFILE是SELECT ... INTO OUTFILE的逆操作，SELECT句法。为了将一个数据库的数据写入一个文件，使用SELECT ... INTO OUTFILE，为了将文件读回数据库，使用LOAD DATA INFILE。
SELECT...INTO OUTFILE 'file_name'形式的SELECT可以把被选择的行写入一个文件中。该文件被创建到服务器主机上，因此您必须拥有FILE权限，才能使用此语法。
输出不能是一个已存在的文件。防止文件数据被篡改。
你需要有一个登陆服务器的账号来检索文件。否则 SELECT ... INTO OUTFILE 不会起任何作用。
在UNIX中，该文件被创建后是可读的，权限由MySQL服务器所拥有。这意味着，虽然你就可以读取该文件，但可能无法将其删除。

-- 查看库表创建语句
以查询Test的建表语句为例：
SHOW CREATE TABLE TEST;
-- 查看当前所在库事件
show events;
-- 查看事件具体代码
show create event [事件名];
-- 创建事件
CREATE EVENT [IF NOT EXIST]  event_name
ON SCHEDULE schedule
DO
event_body

-- 导入数据
mysql> LOAD DATA LOCAL INFILE 'dump.txt' INTO TABLE mytbl
FIELDS TERMINATED BY ':'
LINES TERMINATED BY '\r\n';
-- 在导入数据之前，利用 linux 文本处理脚本把导出的自增字段删掉，再导入
cut -f 2-4 [表名].csv > tmp -- 删除第一列自增字段
sed 's/^/""\t&/g' tmp > tmp2 -- 添加第一列为 "" 空字符串表示自增字段插入
TRUNCATE TABLE [表名]; -- 清空表数据【之前已经删表重建了】
LOAD DATA LOCAL INFILE '/var/lib/mysql-files/tmp2' INTO TABLE [表名]
FIELDS TERMINATED BY '\t' 
ENCLOSED BY '"' 
LINES TERMINATED BY '\n';
SELECT * FROM [表名] LIMIT 5;
```

#### [GROUP BY 多个字段分组](https://segmentfault.com/a/1190000006821331)
- 在mysql 中使用 group by 对表中的数据进行分组时
    + group by x 意思是将所有具有相同 x 字段值的记录放到一个分组里
    + __group by x,y 意思是将具有相同 x 字段值和 y 字段值的记录放到一个分组里__

#### [修改事件EVENT](https://www.cnblogs.com/18JG23/p/6296679.html)
```sql
ALTER EVENT event_name
ON SCHEDULE schedule
ON COMPLETION [NOT] PRESERVE
RENAME TO new_event_name
ENABLE | DISABLE
DO
  event_body
```
#### 修改存储过程
- MySQL __不提供 ALTER 关键字对存储过程代码进行修改__，只提供对存储过程的权限修改，如果要修改存储过程代码，__只能采用 DROP 删掉后重新创建存储过程__
- 参考资料：
    + [MySQL修改存储过程（ALTER PROCEDURE）](http://c.biancheng.net/view/2594.html)
    + [MySQL修改存储过程](https://blog.csdn.net/nangeali/article/details/76285427)
    + [MYSQL修改存储过程](https://blog.csdn.net/lvhuike/article/details/105641347)
    + [MySQL——修改、删除存储过程和函数](https://blog.csdn.net/qq_41573234/article/details/80412581)

```sql
-- 修改存储过程
ALTER {PROCEDURE | FUNCTION} sp_name [characteristic……]
characteristic，表示要修改存储过程的哪个部分，Characteristic的取值如下
CONTAINS SQL，表示子程序包含SQL语句，但是，不包含读或写数据的语句
NO SQL，表示子程序中，不包含SQL语句
READS SQL DATA，表示子程序中，包含读数据的语句
MODIFIES DATA，表示子程序中，包含写数据的语句
SQL SECURITY {DEFINER | INVOKER}，指明谁有权限来执行：DEFINER，表示只有定义者，自己才能够执行；INVOKER，表示调用者可以执行
COMMENT’string’，表示注释信息
-- 删除存储过程
DROP   {  PROCEDURE  |   FUNCTION   }   [  IF EXISTS ]    Sp_name
```

#### 查看数据库表占用磁盘大小
- 参考资料：[Mysql——查看数据库,表占用磁盘大小](https://www.cnblogs.com/linjiqin/p/11446855.html)
```sql
1、查询所有数据库占用磁盘空间大小
select 
TABLE_SCHEMA, 
concat(truncate(sum(data_length)/1024/1024,2),' MB') as data_size,
concat(truncate(sum(index_length)/1024/1024,2),'MB') as index_size
from information_schema.tables
group by TABLE_SCHEMA
ORDER BY data_size desc;

2、查询单个库中所有表磁盘占用大小
select 
TABLE_NAME, 
concat(truncate(data_length/1024/1024,2),' MB') as data_size,
concat(truncate(index_length/1024/1024,2),' MB') as index_size
from information_schema.tables 
where TABLE_SCHEMA = 'xinyar_erp'
group by TABLE_NAME
order by data_length desc;

3、使用optimize命令
optimize table tb_report_inventory;
使用的时间比较长，需要耐心等待。

注意：optimize执行时会将表锁住，所以不要在高峰期使用。也不要经常使用，每月一次就足够了
```
