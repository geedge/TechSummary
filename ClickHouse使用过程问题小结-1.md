## ClickHouse 使用过程问题小结-1
了解 ClickHouse 基本机制及命令，完成建库操作
### [官方文档](https://clickhouse.yandex/docs/zh/)跳读笔记
#### 1. Clickhouse基本介绍
1. ClickHouse是一个用于 __联机分析(OLAP)__ 的 __列式数据库__ 管理系统(DBMS)。常见的行式数据库系统有： MySQL、 Postgres和MS SQL Server。常见的列式数据库有： Vertica、 Paraccel (Actian Matrix， Amazon Redshift)、 Sybase IQ、 Exasol、 Infobright、InfiniDB、 MonetDB (VectorWise， Actian Vector)、 LucidDB、 SAP HANA、 Google Dremel、 Google PowerDrill、Druid、 kdb+。
2. 行式数据库与列式数据库的区别是 __组织存储数据的物理形式__。行式数据库中，一行表示一个数据项，一个数据项中包含多列属性/字段值，__处于同一行中的数据总是被物理的存储在一起__；列式数据库中，一列表示一个数据项，一个数据项中包含多行属性/字段值，列式数据库总是 __将同一列的数据存储在一起， 不同列的数据也总是分开存储__。【看着其实就是转了个置】
3. 在架构设计阶段，__不同的存储方式适合不同的场景__， 这里的查询场景包括： 进行了哪些查询， 多久查询一次以及各类查询的比例； 每种查询读取多少数据————行、 列和字节； 读取数据和写入数据之间的关系； 使用的数据集大小以及如何使用本地的数据集； 是否使用事务,以及它们是如何进行隔离的； 数据的复制机制与数据的完整性要求； 每种类型的查询要求的延迟与吞吐量等等。
4. OLAP场景的关键特征：
    - __大多数是读__请求
    - 数据总是 __以相当大的批(> 1000 rows)进行写入__
    - __不修改__已添加的数据
    - __每次查询都从数据库中读取大量的行， 但是同时又仅需要少量的列__
    - __宽表__， 即每个表包含着大量的列
    - 较少的查询(通常每台服务器每秒数百个查询或更少)
    - 对于简单查询， 允许延迟大约50毫秒
    - __列中的数据相对较小__： 数字和短字符串(例如， 每个URL 60个字节)
    - 处理单个查询时需要高吞吐量（ 每个服务器每秒高达数十亿行）
    - __事务不是必须__的
    - 对数据 __一致性要求低__
    - 每一个查询除了一个大表外都很小
    - __查询结果明显小于源数据__， 换句话说， 数据被过滤或聚合后能够被盛放在单台服务器的内存中
5. __列式数据库更适合于OLAP场景__(对于大多数查询而言， 处理速度至少提高了100倍)
    - 针对分析类查询， 通常只需要读取表的一小部分列。
    - 由于数据总是打包成批量读取的， 所以压缩是非常容易的。 同时数据按列分别存储这也更容易压缩。 这进一步降低了I/O的体积
    - 由于I/O的降低， 这将帮助更多的数据被系统缓存
6. 数据按列存储并且按列执行是很有意义
7. 在一个真正的列式数据库管理系统中， __除了数据本身外不应该存在其他额外的数据__。 这意味着为了避免在值旁边存储它们的长度“number”， 你必须支持固定长度数值类型。如果不是这样的话， 这将对CPU的使用产生强烈影响。 即使是在未压缩的情况下， 紧凑的存储数据也是非常重要的， 因为解压缩的速度主要取决于未压缩数据的大小。
8. ClickHouse不单单是一个数据库， 它是一个数据库管理系统。 因为它 __允许在运行时创建表和数据库、 加载数据和运行查询， 而无需重新配置或重启服务__。
9. 若想达到比较优异的性能， 数据压缩确实起到了至关重要的作用
10. ClickHouse被 __设计用于工作在传统磁盘上__ 的系统， 它提供每GB更低的存储成本， 但如果有可以使用SSD和内存， 它也会合理的利用这些资源。
11. 多服务器分布式处理：在ClickHouse中， 数据可以保存在不同的shard上， 每一个shard都由一组用于容错的replica组成， 查询可以并行地在所有shard上进行处理。 这些对用户来说是透明的
12. 为了高效的使用CPU， 数据不仅仅按列存储， __同时还按向量(列的一部分)进行处理__， 这样可以更加高效地使用CPU
13. ClickHouse __支持在表中定义主键__。 为了使查询能够快速在主键中进行范围查找， __数据总是以增量的方式有序的存储在MergeTree中__。 因此， 数据可以持续不断地高效的写入到表中， 并且写入的过程中不会存在任何加锁的行为。
14. __按照主键对数据进行排序__， 这将帮助ClickHouse在几十毫秒以内完成对数据特定值或范围的查找
15. ClickHouse可以认为是缺点的功能
    - __没有完整的事务支持__
    - __缺少高频率， 低延迟的修改或删除已存在数据__的能力。 仅能用于批量删除或修改数据， 但这符合 GDPR
    - 稀疏索引使得ClickHouse __不适合通过其键检索单行的点查询__
16. 吞吐量可以使用每秒处理的行数或每秒处理的字节数来衡量
17. 在数据没有加载的情况下， 查询所需要的延迟可以通过以下公式计算得知： __查找时间（ 10 ms） * 查询的列的数量 * 查询的数据块的数量__。
18. 在相同的情况下， ClickHouse可以在 __单个服务器上每秒处理数百个查询__（ 在最佳的情况下最多可以处理数千个） 。 但是由于这不适用于分析型场景。 因此我们建议每秒最多查询100次。
19. 建议 __每次写入不少于1000行的批量写入__， 或 __每秒不超过一个写入请求__。为了提高写入性能， 可以 __使用多个INSERT进行并行写入__， 这将带来线性的性能提升
20. Clickhouse应用：允许直接从原始数据中动态查询并生成报告，基于用户定义的字段， 对实时访问、 连接会话， 生成实时的统计报表。 这种需求往往需要复杂聚合方式，比如对访问用户进行去重。 构建报表的数据， 是实时接收存储的新数据。
21. 有一种流行的观点认为， __想要有效的计算统计数据， 必须要聚合数据， 因为聚合将降低数据量__。

#### 2. Clickhouse入门
1. 为CentOS/RedHat __安装__：首先添加官方存储库，$ `sudo yum install yum-utils` ， $ `sudo rpm --import https://repo.yandex.ru/clickhouse/CLICKHOUSE-KEY.GPG` ， $ `sudo yum-config-manager --add-repo https://repo.yandex.ru/clickhouse/rpm/stable/x86_64` ；运行这些命令以实际安装包 $ `sudo yum install clickhouse-server clickhouse-client`
2. __后台启动/停止服务__： `sudo service clickhouse-server start` `sudo ervice clickhouse-server stop`
3. __日志文件目录__： `/var/log/clickhouse-server/ `
4. __配置文件目录__： `/etc/clickhouse-server/config.xml`
5. 控制台中直接启动服务： `clickhouse-server --config-file=/etc/clickhouse-server/config.xml` 。这种情况下， 日志将被打印到控制台中， 这在开发过程中很方便。如果配置文件在当前目录中， 你可以不指定‘--configfile’参数。 它默认使用‘./config.xml’。
6. 使用命令行客户端连接到服务： `clickhouse-client` 。默认情况下它使用‘default’用户无密码的与localhost:9000服务建立连接。 客户端也可以用于连接远程服务 `clickhouse-client --host=example.com`
7. 在ClickHouse中创建表结构：`CREATE TABLE table_name`与MySQL类似，关键在于数据类型、数据库引擎、主键等区别
8. ClickHouse 可以接受多种数据格式， 可以在 ( INSERT ) 以及 ( SELECT ) 请求中使用：常用 `TabSeparated` `CSV` `JSON`
9. __JDBC 驱动__ 有三个：官方提供的、ClickHouse-Native-JDBC、clickhouse4j
10. 还有ODBC驱动和C++客户端库，另外第三方开发的库：支持 Python Java Scala Go
11. 第三方集成库：有 __MySQL__(clickhouse-mysql-data-reader) __Kafka__(clickhouse_sinker) Kubernetes(clickhouse-operator) __Grafana__(clickhouse-grafana) __MaxMind__(clickhouse-maxmind-geoip) 等
12. 编程语言生态系统：有 __Python__(pandahouse) __Java__(clickhouse-hdfs-loader) __Scala__(clickhouse-scala-client) 等
13. 第三方开发的可视化界面：有 Tabix HouseOps LightHouse __DBeaver__(常用) clickhouse-cli clickhouse-flamegraph 等
14. 数据类型：【加粗表示会用到】
    - __整数__（无符号U和有符号） UInt8, __UInt16__, __UInt32__, UInt64, Int8, Int16, Int32, Int64
    - 浮点数 Float32, Float64 NaN and Inf
    - 定点数 Decimal(P, S), Decimal32(S), Decimal64(S), Decimal128(S)
    - 布尔值 Boolean Values 没有单独的类型来存储布尔值。 可以使用 UInt8 类型， 取值限制为 0 或 1
    - __字符串__ __String__ 可以任意长度，包含任意的字节集，包含空字节。 
    - 定长字符串 FixedString(N) 固定长度 N 的字符串（ N 必须是严格的正自然数） ，如二进制表示的IP地址（ IPv6使用 FixedString(16) ）如果字符串包含的字节数少于`N',将对字符串末尾进行空字节填充。如果字符串包含的字节数大于 N ,将抛出 Too large value for FixedString(N) 异常。__当做数据查询时， ClickHouse不会删除字符串末尾的空字节。 如果使用 WHERE 子句， 则须要手动添加空字节以匹配 FixedString 的值__。 
    - __UUID__ A universally unique identifier (UUID) is a 16-byte number used to identify records.  如 61f0c404-5cb3-11e7-907b-a6006ad3dba0
    - __日期__ __Date__ 日期类型， 用两个字节存储， 表示从 1970-01-01 (无符号) 到当前的日期值。 日期中没有存储时区信息
    - __时间戳__ __DateTime__ 时间戳类型。 用四个字节（ 无符号的） 存储 Unix 时间戳） 。 允许存储与日期类型相同的范围内的值。 最小值为 0000-00-00 00:00:00。 时间戳类型值精确到秒（ 没有闰秒） 。
    - 枚举 Enum8, Enum16 Enum 保存 'string'= integer 的对应关系。
    - 数组 Array(T) 由 T 类型元素组成的数组。
    - __元组__ Tuple(T1, T2, ...) 其中每个元素都有单独的 类型。
    - Nullable(TypeName) 允许用特殊标记 (NULL) 表示"缺失值"， 可以与 TypeName 的正常值存放一起。要在表的列中存储 Nullable 类型值， ClickHouse 除了使用带有值的普通文件外， 还使用带有 NULL 掩码的单独文件。注意点 使用 Nullable 几乎总是对性能产生负面影响， 在设计数据库时请记住这一点
    - 嵌套数据结构 Nested(Name1 Type1, Name2 Type2, ...) 嵌套数据结构类似于嵌套表。 嵌套数据结构的参数（ 列名和类型） 与 CREATE 查询类似。
    - 特殊数据类型 Special Data Types 特殊数据类型的值既不能存在表中也不能在结果中输出， 但可用于查询的中间结果
        + Expression 用于表示高阶函数中的Lambd表达式。
        + Set 可以用在 IN 表达式的右半部分
        + Nothing 此数据类型的唯一目的是表示不是期望值的情况。 所以不能创建一个 Nothing 类型的值
        + __Interval__ The family of data types representing time and date intervals. The resulting types of theINTERVAL operator. Supported interval types: SECOND MINUTE HOUR DAY WEEK MONTH QUARTER YEAR
    - Domains Domain类型是特定实现的类型， 它总是与某个现存的基础类型保持二进制兼容的同时添加一些额外的特性， 以能够在维持磁盘数据不变的情况下使用这些额外的特性。 目前ClickHouse暂 __不支持自定义domain类型__。
    - __IPv4__ 是与 UInt32 类型保持二进制兼容的Domain类型， 其用于存储IPv4地址的值。 它提供了更为紧凑的二进制存储的同时支持识别可读性更加友好的输入输出格式。也可以使用 IPv4 类型的列作为主键。在写入与查询时， IPv4 类型能够识别可读性更加友好的输入输出格式，提供更为紧凑的二进制存储格式。不可隐式转换为除 UInt32 以外的其他类型类型。 如果要将 IPv4 类型的值转换成字符串， 你可以使用 IPv4NumToString() 显示的进行转换，或可以使用 CAST 将它转换为 UInt32 类型。
    - __IPv6__ 是与 FixedString(16) 类型保持二进制兼容的Domain类型， 其用于存储IPv6地址的值。 它提供了更为紧凑的二进制存储的同时支持识别可读性更加友好的输入输出格式，同上类似，不可隐式转换为除 FixedString(16) 以外的其他类型类型。
15. 数据库引擎：使用的所有表都是由数据库引擎所提供的。默认情况下， ClickHouse使用自己的数据库引擎， 该引擎提供可配置的表引擎和所有支持的SQL语法。除此之外，还可以选择使用以下的数据库引擎：
    - MySQL引擎 __用于将远程的MySQL服务器中的表映射到ClickHouse中__ ， 并允许您对表进行 INSERT 和 SELECT 查询， 以方便您在ClickHouse与MySQL之间进行数据交换。MySQL 数据库引擎会将对其的查询转换为MySQL语法并发送到MySQL服务器中， 因此您可以执行诸如 SHOW TABLES 或 SHOW CREATE TABLE 之类的操作。但 __无法对其执行以下操作__ ：ATTACH / DETACH、DROP、RENAME、CREATE、TABLE、ALTER
16. 表引擎：表引擎（ 即表的类型） 决定了，数据的存储方式和位置， 写到哪里以及从哪里读取数据；支持哪些查询以及如何支持；并发数据访问；索引的使用（ 如果存在） ；是否可以执行多线程请求；数据复制参数。ClickHouse __不要求主键惟一__。 所以， 你可以插入多条具有相同主键的行。指定一个跟排序键（ 用于排序数据片段中行的表达式） 不一样的主键（ 用于计算写到索引文件的每个标记值的表达式） 是可以的。 这种情况下， __主键表达式元组必须是排序键表达式元组的一个前缀__。
    - __MergeTree__ 适用于高负载任务的最通用和功能最强大的表引擎。 这些引擎的共同特点是 __可以快速插入数据并进行后续的后台数据处理__ 。MergeTree系列引擎支持数据复制（ 使用Replicated* 的引擎版本） ， 分区和一些其他引擎不支持的其他功能。该类型的引擎： MergeTree ReplacingMergeTree SummingMergeTree AggregatingMergeTree CollapsingMergeTree VersionedCollapsingMergeTree * GraphiteMergeTree
        + __存储的数据按主键排序__：可以创建一个用于快速检索数据的小稀疏索引。默认情况下主键跟排序键（ 由 ORDER BY 子句指定） 相同。 
        + 允许使用分区：ClickHouse 中某些带分区的操作会比普通操作更快。 查询中指定了分区键时 ClickHouse 会自动截取分区数据。 这也有效增加了查询性能
        + 支持数据副本
        + 支持数据采样
    - Log 具有最小功能的轻量级引擎。 当您需要快速写入许多小表（ 最多约100万行） 并在以后整体读取它们时， 该类型的引擎是最有效的。该类型的引擎 TinyLog StripeLog Log
    - Intergation engines 用于 __与其他的数据存储与处理系统集成__ 的引擎。该类型的引擎 __Kafka__ __MySQL__ ODBC __JDBC__ HDFS
    - 用于其他特定功能的引擎 Distributed File Set URL View 等
17. 列和表的TTL TTL可以设置值的生命周期， 它既可以为整张表设置， 也可以为每个列字段单独设置。 如果 TTL 同时作用于表和字段， ClickHouse会使用先到期的那个。被设置TTL的表， 必须拥有Date 或 DateTime 类型的字段。 要定义数据的生命周期， 需要在这个日期字段上使用操作符。
    - 列字段 TTL 当列字段中的值过期时, ClickHouse会将它们替换成数据类型的默认值。TTL 子句不能被用于主键字段。
    - __表 TTL__ 当表内的数据过期时, ClickHouse会删除所有对应的行。
18. 数据副本：只有 MergeTree 系列里的表可支持副本。副本是表级别的， 不是整个服务器级的。 所以， 服务器里可以同时有复制表和非复制表。副本不依赖分片。 每个分片有它自己的独立副本。__对于 INSERT 和 ALTER 语句操作数据的会在压缩的情况下被复制 。而 CREATE ， DROP ， ATTACH ， DETACH 和 RENAME 语句只会在单个服务器上执行， 不会被复制。__要使用副本， 需在配置文件中设置 ZooKeeper 集群的地址。 需要 ZooKeeper 3.4.5 或更高版本。复制是多主异步。 INSERT 语句（ 以及 ALTER ） 可以发给任意可用的服务器。 数据会先插入到执行该语句的服务器上， 然后被复制到其他服务器。
19. 创建复制表：在表引擎名称上加上 Replicated 前缀。 例如： ReplicatedMergeTree 。
20. 数据处理：当数据被插入到表中时， 他们将被原样保存。 ClickHouse 定期合并插入的数据片段， 并在这个时候 __对所有具有相同主键的行中的列进行汇总__， 将这些行替换为包含汇总数据的一行记录。

### 代码
```sql
# 建库
CREATE TABLE [IF NOT EXISTS] [db.]table_name [ON CLUSTER cluster]
(
name1 [type1] [DEFAULT|MATERIALIZED|ALIAS expr1],
name2 [type2] [DEFAULT|MATERIALIZED|ALIAS expr2],
...
INDEX index_name1 expr1 TYPE type1(...) GRANULARITY value1,
INDEX index_name2 expr2 TYPE type2(...) GRANULARITY value2
) ENGINE = MergeTree()
[PARTITION BY expr]
[ORDER BY expr]
[PRIMARY KEY expr]
[SAMPLE BY expr]
[SETTINGS name=value, ...]

TTL date_time + INTERVAL 1 MONTH
TTL date_time + INTERVAL 15 HOUR
```
```bash
# 导出json格式数据
clickhouse-client -h 127.0.0.1 --port 9001 -m -u username --password xxxxx -q "SELECT * FROM 
database.table WHERE found_time > toUnixTimestamp('2019-11-"$i" 00:00:00'
) and found_time < toUnixTimestamp('2019-11-"$[$i+1]" 00:00:00') FORMAT JSON" > 2019-11-$i.json
```

### 问题
#### clickhouse显示接收数据超时
导出数据的时候，数据量太大， 导出时间过长超过配置限制，导致导出中断出错，clickhouse显示接收数据超时。依次找到如下解决方案，最终修改了配置文件中的最长执行时间 max_execution_time 参数，才搞定

1. [超时解决方案](https://github.com/ClickHouse/ClickHouse/issues/2833)：Dirty workaround: as suggested above (#2833 (comment)), add following options in DSN &send_timeout=30000&receive_timeout=3000 (at least, works for me). It looks like the data for INSERT queries was not sent for more than 300 seconds. (Probably there was some data, then a pause for more than 300 seconds.) You should consult with author of the library: @kshvakov Also you can increase timeouts: it is send_timeout and receive_timeout these settings should be placed in users.xml in yandex/profiles/default. 
2. 参考上面，在 vim /etc/clickhouse-server/users.xml 中，添加了 send_timeout 和 receive_timeout 的标签`<send_timeout>30000</send_timeout>` `<receive_timeout>3000</receive_timeout>`——没用
3. [修改系统配置后需要重启服务，用户配置不用](https://www.lubaogui.com/2019/03/06/clickhouse（二）运维/)：`service clickhouse-server stop ` `service clickhouse-server start` ——没用
4. 试了下，和上面配置的那两个没有关系，关键是师兄托人改的 users.xml 里面的那个 7200 `<max_execution_time>7200</max_execution_time>` 最长执行时间，改成现在是 2h 才行
