## MySQL 使用过程问题小结
### 数据库设计&使用规范
#### 数据库&表&字段命名规范
[参考摘抄](https://www.cnblogs.com/slgkaifa/p/7245519.html)

1. 表名一般以 __模块名称_详细表名__ 来实现，同一个模块的前缀是一样的
2. 表名称不应该取得太长（ __一般不超过三个英文单词__ ，不推荐使用中文拼音，总长度不要超过30个字符）
3. 一些作为 __多对多连接表__ ，能够 __使用两个表的前缀作为表名__ ：如：用户登录表User_Login。用户分组表User_GroupInfo，这两个表建立多对多关系的表名为：User_Group_Relation（关系统一用Relation）
4. 注意一点， __主键在做其它表的外键时__ ，或者在被其它表引用时， __字段说明和字段名尽量保持一致__ 。比方发帖表BBS_Topic里的用户字段写成UI_ID，这样跟用户信息表User_Info的主键UI_ID保持一致，看起来舒服，相应关系非常明白，也不容易错，前后不一致时easy令人费解。
5. 当系统中有一些少量的、反复出现的值时， __使用字典表来节约存储空间和优化查询__ 。如地区、系统中用户类型的代号等。这类值不会在程序的执行期变化，可是须要存储在数据库中。 __一般数据库中都有一个数据字典表__ ，用来保存系统所用到的基础数据，大型的字段表如省份城市区域的字典表， __统一以Dictionary_作为前缀__ 。建一个基础数据字典表的作用非常重要，其它的比方地区表(Sys_DicArea)，汉语拼音表（Sys_DicCharacter）(用来汉字和拼音的转换)由于数据量较大，单独建表。
6. 与字段有关， __默认的一些特殊字段__ 。比方一些业务处理表中，除了加入生成的自己主动编号ID（一般作为主键用），该记录创建的时间CreateDate(创建时间)、该记录的创建人CreatBy（注意这里没UI_ID（用户信息表User_Info的主键UI_ID）,由于还有改动人），最后改动人LastEditBy。最后改动时间LastEditDate。（这些能够直接使用中文字符，而不使用编码，提高查询的效率）同一时候有的时候须要注意， __删除的时候并不真的删除该记录__ ，而是 __加入一个标识位__ ，比方XX_DeleteStaus删除状态。1是有效的，0则是无效的。
7. __命名表时，用单数形式表示名称__ 。比如。使用 Employee，而不是 Employees。
8. 数据库中应建立这样一个表，就是 __数据库本身的字段信息__ 。表的说明，也就是数据库设计文档的一个表。方便查询使用，有什么不明的能够直接从数据库查询，数据库文档丢失，凝视丢失，都能够又一次起作用。
9. __每一个表都应该有一个主键，这个主键最好是数字，并且是递增的__ 。有非常多表的主键用32位字符编码，这样做的目的很多其它的是从安全考虑的。由于字符多时索引时效率低，而使用自增列也不是非常少，比方加入主表和从表操作时。主表的主键是从表的外键，这个时候还有取返回值，然后再加入，不能够同一时候加入。主键能够用自己定义的规则，大部分用MAX（ID）的做法，也能够自己定义一个序列表，有点像序列。或者用时间的年月日秒详细到毫秒。关于列的命名,建议对数据类型也做一些规范，由于非常easy确定。仅仅有四种主要类型：数字，字符，时间，逻辑值，这些在类型上和长度上都能够定好规范，统一起来。
10. __操作日志表，登录日志表，这是数据库中必备的两个表__ 。这个记录也须要做进一步的保存。这个有两种情形，一是详细到单个字段的操作日志。二是整个表的操作日志。
常见的几个表详细说明：操作日志表Sys_OperateLog、登录日志表Sys_LoginLog、系统字典表Sys_Dictionary、系统字典表类型Sys_DicType

[参考摘抄](https://www.cnblogs.com/pangguoming/p/7126512.html)

1. 数据库命名规范：采用 __26个英文字母__ (区分大小写)和0-9的自然数(经常不需要)加上 __下划线\___ 组成，命名简洁明确，多个单词用下划线_分隔，一个项目一个数据库，多个项目慎用同一个数据库
2. 数据表命名规范：__全部小写命名，禁止出现大写__； __禁止使用数据库关键字__ ，如：name，time ，datetime，password等；表的名称一般 __使用名词或者动宾短语__ ；明细表的名称为：主表的名称+字符dtl（detail缩写）；表必须 __填写描述信息__ （使用SQL语句建表时）。命名规范：①模块_+功能点  示例：alllive_log   alllive_category；②功能点  示例：live   message；③通用表  示例：all_user。统一命名规则， __动宾短语分离且动宾逻辑顺序统一__ 。
3. 数据库字段命名规范：在命名表的列时， __不要重复表的名称__ ；不要在列的名称中包含数据类型；字段命名 __使用完整名称__ ，禁止缩写
4. 所有字段在设计时，除以下数据类型timestamp、image、datetime、smalldatetime、uniqueidentifier、binary、sql_variant、binary 、varbinary外， __必须有默认值__ ， __字符型的默认值为一个空字符值串''，数值型的默认值为数值0，逻辑型的默认值为数值0__ ，系统中所有逻辑型中数值0表示为“假”，数值1表示为“真”，datetime、smalldatetime类型的字段没有默认值，必须为NULL
5. 用尽量少的存储空间来存储一个字段的数据：使用int就不要使用varchar、char；用varchar(16)就不要使varchar(256)；IP地址使用int类型；固定长度的类型最好使用char，例如：邮编(postcode)；能使用tinyint就不要使用smallint，int；最好给每个字段一个默认值，最好不能为null；字符转化为数字(能转化的最好转化，同样节约空间、提高查询性能)；避免使用NULL字段(NULL字段很难查询优化、NULL字段的索引需要额外空间、NULL字段的复合索引无效)；少用text类型(尽量使用varchar代替text字段)
6. 数据库中每个字段的规范描述：(1)尽量遵守第三范式的标准(3NF)，表内的每一个值只能被表达一次，表内的每一行都应当被唯一的标识，表内不应该存储依赖于其他键的非键信息；(2)如果字段事实上是与其它表的关键字相关联而未设计为外键引用，需建索引；(3)如果字段与其它表的字段相关联，需建索引；(4)如果字段需做模糊查询之外的条件查询，需建索引；(5)除了主关键字允许建立簇索引外，其它字段所建索引必须为非簇索引；

#### SQL语言编码规范
1. 所有关键字必须大写，如：INSERT、UPDATE、DELETE、SELECT及其子句，IF……ELSE、CASE、DECLARE等
2. 所有函数及其参数中除用户变量以外的部分必须大写
3. 在定义变量时用到的数据类型必须小写

#### 注释编写规范
注释可以包含在批处理中，在触发器、存储过程中包含描述性注释将大大增加文本的可读性和可维护性，本规范建议： 

1. 注释以英文为主，实际应用中，发现以中文注释的SQL语句版本在英文环境中不可用，为避免后续版本执行过程中发生某些异常错误，建议使用英文注释
2. 注释尽可能详细、全面创建每一数据对象前，应具体描述该对象的功能和用途，传入参数的含义应该有所说明，如果取值范围确定，也应该一并说明，取值有特定含义的变量（如boolean类型变量），应给出每个值的含义
3. 注释语法：单行注释、多行注释；单行注释：注释前有两个连字符（--）对变量、条件子句可以采用该类注释；多行注释：符号之间的内容为注释内容，对某项完整的操作建议使用该类注释
4. 注释简洁，同时应描述清晰
5. 函数注释： 编写函数文本--如触发器、存储过程以及其他数据对象--时，必须为每个函数增加适当注释，该注释以多行注释为主

### 数据库安装&登录&退出
1. 数据库安装包下载及安装：

        ```bash
        rpm -i mysql-community-release-el7-5.noarch.rpm
        yum install mysql-community-server
        ```

2. mysql 服务启动停止查看：

        ```bash
        systemctl start mysql # 启动 mysql 
        systemctl stop mysql # 停止 mysql
        systemctl status mysql # 查看 mysql 状态
        ```

3. 安装后初始配置（建立数据库登录用户及密码）：`mysqladmin -u root password`

        ```bash
        mysqladmin -u root password # 新建用户名及密码
        chkconfig --level 2345 mysqld on # 设置 mysql 服务的运行级信息
        systemctl start mysql # 重启 mysql 服务

        [root@iz2ze8e1dj06n0wjqj19n2z ~]# systemctl start mysql
        [root@iz2ze8e1dj06n0wjqj19n2z ~]# mysqladmin -u root password
        New password: 【此处输入密码】
        Confirm new password: 【此处重复输入密码确认】

        [root@iz2ze8e1dj06n0wjqj19n2z ~]# chkconfig --level 2345 mysqld on
        Note: Forwarding request to 'systemctl enable mysqld.service'.
        ```

4. 初始配置后正常运行/登录mysql：`mysql -u root -p`

        ```bash
        mysql -u root -p
        【此处输入密码】
        [root@iz2ze8e1dj06n0wjqj19n2z ~]# mysql -u root -p
        Enter password: 
        Welcome to the MySQL monitor.  Commands end with ; or \g.
        Your MySQL connection id is 2
        Server version: 5.6.43 MySQL Community Server (GPL)

        Copyright (c) 2000, 2019, Oracle and/or its affiliates. All rights reserved.

        Oracle is a registered trademark of Oracle Corporation and/or its
        affiliates. Other names may be trademarks of their respective
        owners.

        Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

        mysql> 
        ```

5. 退出 mysql 命令行界面：`exit` 回车和 `ctrl+d`

        ```bash
        mysql> exit
        Bye
        mysql> ^DBye
        ```

### 数据库表操作&管理
1. 查看数据库列表：`show databases;`

        ```sql
        mysql> show databases;
        +--------------------+
        | Database           |
        +--------------------+
        | information_schema |
        | mysql              |
        | performance_schema |
        +--------------------+
        3 rows in set (0.00 sec)

        mysql> 
        ```

2. 创建数据库：`CREATE DATABASE [数据库名];`

        ```sql
        mysql> CREATE DATABASE dns;
        Query OK, 1 row affected (0.00 sec)
        ```

3. 使用数据库：`use [数据库名];`

        ```sql
        mysql> use dns;
        Database changed
        ```

4. 查看数据库的表列表：`show tables;`

        ```sql
        mysql> show tables;
        Empty set (0.01 sec)
        ```

5. 创建用户：`CREATE USER '[用户名]'@'localhost' IDENTIFIED BY '[用户密码]';`

        ```sql
        mysql> CREATE USER 'root'@'localhost' IDENTIFIED BY '111111';
        Query OK, 0 rows affected (0.01 sec)

        [root@iz2ze8e1dj06n0wjqj19n2z ~]# mysql -u root -p
        Enter password: 111111
        ```

6. 给用户赋权：`GRANT CREATE,DROP,ALTER,INSERT,UPDATE,SELECT,DELETE,INDEX ON [数据库名].* TO '[用户名]'@'localhost';`

        ```sql
        mysql> GRANT CREATE,DROP,ALTER,INSERT,UPDATE,SELECT,DELETE,INDEX ON dns.* TO 'root'@'localhost';
        Query OK, 0 rows affected (0.01 sec)
        mysql> FLUSH PRIVILEGES; # 刷新权限
        Query OK, 0 rows affected (0.01 sec)
        ```

7. 创建数据库表：`CREATE TABLE [表名]([字段1], [字段2], [字段3]……);`

        ```sql
        CREATE TABLE employee (
        employee_id INTEGER UNSIGNED NOT NULL AUTO_INCREMENT,
        last_name VARCHAR(30) NOT NULL,
        first_name VARCHAR(30) NOT NULL,
        email VARCHAR(100) NOT NULL,
        hire_date DATE NOT NULL,
        notes MEDIUMTEXT,
        PRIMARY KEY (employee_id),
        INDEX (last_name),
        UNIQUE (email)
        )
        ENGINE=InnoDB
        DEFAULT CHARSET = utf8; # 修改数据库表默认编码方式
        # 遇到问题：修改编码格式后，UNIQUE的那个字段长度太长，超过索引支持
        ```

8. 修改数据库&表默认编码：[参考](https://www.cnblogs.com/shootercheng/p/5836657.html )

        ```sql
        mysql> show variables like 'character_set_database';
        +------------------------+--------+
        | Variable_name          | Value  |
        +------------------------+--------+
        | character_set_database | latin1 |
        +------------------------+--------+
        1 row in set (0.00 sec)

        create database <数据库名> character set utf8; # 创建数据库时添加字符编码设置
        create table tb_books (
            name varchar(45) not null,
            price double not null,
            bookCount int not null,
            author varchar(45) not null ) default charset = utf8; # 创建表时添加字符编码设置

        alter database <数据库名> character set utf8; # 修改数据库字符编码
        alter table <表名> character set utf8; # 修改表字符编码
        alter table <表名> change <字段名> <字段名> <类型> character set utf8; # 修改字段字符编码【这里就是要重写两遍字段名】
        alter table user change username username varchar(20) character set utf8 not null;
        # PS：修改编码格式时，若仅修改数据库编码，对于已经建好的表，编码格式不会改变，只对新建的表有用；所以对于已经建好的表，一定要修改到字段才可以，仅修改数据库和表还不行
        ```

9. 添加&修改&删除唯一约束：[MySQL 唯一约束 (UNIQUE KEY)](http://c.biancheng.net/view/2445.html)；[MySQL 创建表多个字段的唯一约束](https://blog.csdn.net/john1337/article/details/84141087)

        ```sql
        # 1. 在创建表时设置唯一约束：在定义完列之后直接使用 UNIQUE 关键字指定唯一约束，语法规则：
        # <字段名> <数据类型> UNIQUE
        mysql> CREATE TABLE tb_dept2
            -> (
            -> id INT(11) PRIMARY KEY,
            -> name VARCHAR(22) UNIQUE, # 唯一约束
            -> location VARCHAR(50)
            -> );
        # 2. 在修改表时添加唯一约束：在修改表时添加唯一约束的语法格式为：
        # ALTER TABLE <数据表名> ADD CONSTRAINT <唯一约束名> UNIQUE(<列名>);
        mysql> ALTER TABLE tb_dept1
            -> ADD CONSTRAINT unique_name UNIQUE(name);
        # 3. 删除唯一约束：在 MySQL 中删除唯一约束的语法格式如下：
        # ALTER TABLE <表名> DROP INDEX <唯一约束名>;
        mysql> ALTER TABLE tb_dept1
            -> DROP INDEX unique_name;
        # 4. 多字段组合唯一约束，mysql 提供了两种方式：
        alter table xx add unique key no_account(no,collection_account) # unique key
        alter table xx add unique no_account_index(no,collection_account); # unique index
        ```

10. 唯一约束长度太长导致建表出错：`修改 mysql 全局配置并删表重建添加表动态属性`；参考[MySQL唯一约束长度限制](https://blog.51cto.com/13120271/2315189)；参考[修改Mysql索引长度限制](https://blog.csdn.net/u010719917/article/details/70598813)：对row_format为dynamic格式 ,可以指定索引列长度大于767 bytes。但是索引列总长度的不能大于3072 bytes的限制仍然存在；参考[mysql改变innodb_large_prefix](http://www.voidcn.com/article/p-uoferwcl-bwa.html) 默认约束长度为 767 字节，来源是因为之前 utf8 256*3字节-1 = 767 字节。可以通过修改 innodb_large_prefix 参数扩展至 3072 字节，文章说：一般情况下不建议使用这么长的索引，对性能有一定影响；最后这篇文章提出来的最终版原因是：MySQL 索引只支持767个字节，utf8mb4 每个字符占用4个字节，所以索引最大长度只能为191个字符，即varchar(191)，若想要使用更大的字段，mysql需要设置成支持数据压缩，并且修改表属性 row_format ={DYNAMIC|COMPRESSED}。最后搞定顺序：1. 在 MySQL 命令行中：`SET GLOBAL innodb_file_format = Barracuda;`, `SET GLOBAL innodb_file_per_table = ON;`, `SET GLOBAL  innodb_large_prefix=on;`，修改全局配置；2. 然后删除需要建索引的表后，重建表在最后添加：`ROW_FORMAT = DYNAMIC; – 或COMPRESSED(在CREATE结束时)`

        ```sql
        mysql> show variables like 'innodb_large_prefix';
        +---------------------+-------+
        | Variable_name       | Value |
        +---------------------+-------+
        | innodb_large_prefix | OFF   |
        +---------------------+-------+
        1 row in set (0.00 sec)

        mysql> set global innodb_large_prefix=on; # 修改全局配置
        Query OK, 0 rows affected (0.00 sec)

        mysql> show variables like 'innodb_large_prefix';
        +---------------------+-------+
        | Variable_name       | Value |
        +---------------------+-------+
        | innodb_large_prefix | ON    |
        +---------------------+-------+
        1 row in set (0.00 sec)

        CREATE TABLE `domain` (
        ……
        ) ENGINE=InnoDB  DEFAULT CHARSET=utf8mb4 ROW_FORMAT=DYNAMIC; # 创建表添加字段
        ```

11. 清空数据表中数据：`delete from [表名];` 或者 `truncate table [表名];`  [TRUNCATE 和 DELETE 的区别](https://blog.csdn.net/stail111/article/details/5718771)不带where参数的delete语句可以删除mysql表中所有内容，使用truncate table也可以清空mysql表中所有内容；效率上truncate比delete快，但truncate删除后不记录mysql日志，不可以恢复数据；delete的效果有点像将mysql表中所有记录一条一条删除到删完，保留mysql表的结构；而truncate相当于重新创建了这个表，所有的状态都相当于新表。

        ```sql
        delete from 表名;
        truncate table 表名;
        ```

12. 删除数据库&表：`drop database [库名];` 和 `DROP TABLE [表名];`

        ```sql
        DROP TABLE table_name ;

        mysql> drop database dns;
        Query OK, 6 rows affected (0.36 sec)

        mysql> show databases;
        +--------------------+
        | Database           |
        +--------------------+
        | information_schema |
        | mysql              |
        | performance_schema |
        +--------------------+
        3 rows in set (0.01 sec)
        ```
