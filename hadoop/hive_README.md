# hadoop systems

Hive by nature is a data warehousing tools based on hadooop. so it converts the mapreduce (or spark) computing and manage the data in hdfs(s3)
into a SQL like warehouse management. reduce the complexity for the distributed systems, and it's a structured distributed data warehouse entrance 
on the hadoop. 
it's mainly use as a distributed table mapping and query tools. modify and update, delete seldom uses it. 
### hive data type DDL
* TYNYINT  : byte (1 bytes)
* SMALLINT : short (2 bytes)
* INT: int(4bytes)
* BIGINT: long(8bytes)
* BOOLEAN: boolean (true or false)
* float: float (single)
* DOUBLE: double(double)
* STRING: string (charset)
* TIMESTAMP: 
* BINARY: bytes 
complex type: 
* array
* struct 
* mapping 

CREATE [EXTERNAL] TABLE IF NOT EXIST test 
(name STRING,
 age TINYINT,
 payments FLOAT Comment "payments should be 0.00 precision",
 address struct<stree:string>,city:string>,
 children map<string, int>,
 friends array<string> comment 'friends are the users that you want to linked to ,a really really really long comment should have line break'
 ) comment 'this is the table comment ' 
 partitioned by (name string, age int) # partition tables 
 clustered by (colname, col_name2)  # bucket tables
 sorted by (col_name asc|desc ) into num_buckets BUCKETS
 row format delimited fields terminated by ',' 
 collection items terminated by '_' 
 map keys terminated by ':'  
 lines terminated by '\n'
 stored as file_format (：TEXTFILE|SEQUENCEFILE|RCFILE)
 location hdfs_path
tblproperties(key1=value1, key2=value2)
as select statement;
如果压缩，使用 SEQUENCEFILE

1.  ALTER TABLE table_name RENAME TO new_table_name;
2. ALTER TABLE table_name CHANGE [COLUMN] col_old_name col_new_name column_type
[COMMENT col_comment] [FIRST|AFTER column_name]
3. ALTER TABLE table_name ADD|REPLACE COLUMNS (col_name data_type [COMMENT
col_comment], ...)

# test env install
to install hive on kubernetes, could use the s3 and hive on spark. 

1. install directpv: https://github.com/minio/directpv
```bash
kubectl krew install directpv

kubectl directpv install

kubectl directpv info

kubectl directpv discover
```


# hive optimization

Optimizing Hive queries is crucial for achieving better performance and scalability in a data warehouse environment. Here are some tips and best practices for optimizing Hive queries:


多次INSERT单次扫描表
默认情况下，Hive会执行多次表扫描。因此，如果要在某张hive表中执行多个操作，建议使用一次扫描并使用该扫描来执行多个操作。

比如将一张表的数据多次查询出来装载到另外一张表中。如下面的示例，表my_table是一个分区表，分区字段为dt，如果需要在表中查询2个特定的分区日期数据，并将记录装载到2个不同的表中。

INSERT INTO temp_table_20201115 SELECT * FROM my_table WHERE dt ='2020-11-15';
INSERT INTO temp_table_20201116 SELECT * FROM my_table WHERE dt ='2020-11-16';
在以上查询中，Hive将扫描表2次，为了避免这种情况，我们可以使用下面的方式：

FROM my_table
INSERT INTO temp_table_20201115 SELECT * WHERE dt ='2020-11-15'
INSERT INTO temp_table_20201116 SELECT * WHERE dt ='2020-11-16'
这样可以确保只对my_table表执行一次扫描，从而可以大大减少执行的时间和资源。

### table optimization
1. use int instead of the string to improve the performance.
2. try to avoid complex logic in a SQL, and separated them into several tasks.
3. avoid fulltable scann: 
before: 
SELECT * FROM table WHERE date = '2021-01-01' AND region = 'A';
after: 
SELECT * FROM table WHERE partition_date = '2021-01-01' AND partition_region = 'A';

before: 
SELECT * FROM table WHERE region = 'A' AND status = 'ACTIVE';
after：
CREATE INDEX idx_region_status ON table (region, status);
SELECT * FROM table WHERE region = 'A' AND status = 'ACTIVE';

before：
select a.*,b.* from a join b  on a.name=b.name where a.age>30
after：
SELECT a.*, b.* FROM ( SELECT * FROM a WHERE age > 30 ) a JOIN b ON a.name = b.name


优化前：

select count(distinct uid) from test where ds='2020-08-10' and uid is not null
优化后：

select count(a.uid) from (select uid from test where uid is not null and ds = '2020-08-10' group by uid) a


设置自动选择Mapjoin

set hive.auto.convert.join = true; 默认为true
--大表小表的阈值设置（默认25M以下认为是小表）：

set hive.mapjoin.smalltable.filesize=25000000;



3.9 使用with as


大表Join小表

在编写具有Join操作的查询语句时，有一项重要的原则需要遵循：应当将记录较少的表或子查询放置在Join操作符的左侧。这样做有助于减少数据量，提高查询效率，并有效降低内存溢出错误的发生概率。

如果未指定MapJoin，或者不符合MapJoin的条件，Hive解析器将会将Join操作转换成Common Join。这意味着Join操作将在Reduce阶段完成，由此可能导致数据倾斜的问题。为了避免这种情况，可以通过使用MapJoin将小表完全加载到内存中，并在Map端执行Join操作，从而避免将Join操作留给Reducer阶段处理。这种策略有效地减少了数据倾斜的风险。

3.11 大表Join大表

3.11.1 空key过滤

有时候，连接操作超时可能是因为某些key对应的数据量过大。相同key的数据被发送到相同的reducer上，由此导致内存不足。在这种情况下，我们需要仔细分析这些异常的key。通常，这些key对应的数据可能是异常的，因此我们需要在SQL语句中进行适当的过滤。

3.11.2 空key转换

当某个key为空时，尽管对应的数据很丰富，但并非异常情况。在执行join操作时，这些数据必须包含在结果集中。为实现这一目的，可以考虑将表a中那些key为空的字段赋予随机值，以确保数据能够均匀、随机地分布到不同的reducer上。

3.12 避免笛卡尔积

在执行join操作时，若不添加有效的on条件或者使用无效的on条件，而是采用where条件，可能会面临关联列包含大量空值或者重复值的情况。这可能导致Hive只能使用一个reducer来完成操作，从而引发笛卡尔积和数据膨胀问题。因此，在进行join时，务必注意确保使用有效的关联条件，以免由于数据的空值或重复值而影响操作性能。

优化案例

优化前：

SELECT * FROM A, B; 
--在优化前的SQL代码中，使用了隐式的内连接（JOIN），没有明确指定连接条件，导致产生了笛卡尔积

优化后；

SELECT * FROM A CROSS JOIN B;
在优化后的SQL代码中，使用了明确的交叉连接（CROSS JOIN），确保只返回A和B表中的所有组合，而不会产生重复的行。 通过明确指定连接方式，可以避免不必要的笛卡尔积操作，提高查询效率。



SET hive.auto.convert.join=true; --  hivev0.11.0之后默认true
SET hive.mapjoin.smalltable.filesize=600000000; -- 默认 25m
SET hive.auto.convert.join.noconditionaltask=true; -- 默认true，所以不需要指定map join hint
SET hive.auto.convert.join.noconditionaltask.size=10000000; -- 控制加载到内存的表的大小


### Partitioning:

Partitioning your data can significantly improve query performance by reducing the amount of data scanned during query execution.
Partition your tables based on commonly filtered columns, such as date or category.
Use static partitioning for columns with a limited number of distinct values and dynamic partitioning for columns with high cardinality.
Consider using partitioned tables for time-series data to improve query performance for date-range queries.

reduce the amount of data to handle : 
use the partition key: and where to filter and the colum selection all the time. 
CREATE TABLE table_name (
    col1 data_type,
    col2 data_type)
PARTITIONED BY (partition1 data_type, partition2 data_type,….);

### Bucketing:

Bucketing distributes data into a fixed number of buckets based on the hash value of one or more columns.
Use bucketing to distribute data across files and improve data locality evenly.
Choose the number of buckets wisely based on the size of your data and the available resources.
Bucketing is particularly useful for optimizing join operations and aggregations.

分桶表
通常，当很难在列上创建分区时，我们会使用分桶，比如某个经常被筛选的字段，如果将其作为分区字段，会造成大量的分区。在Hive中，会对分桶字段进行哈希，从而提供了中额外的数据结构，进行提升查询效率。

与分区表类似，分桶表的组织方式是将HDFS上的文件分割成多个文件。分桶可以加快数据采样，也可以提升join的性能(join的字段是分桶字段)，因为分桶可以确保某个key对应的数据在一个特定的桶内(文件)，所以巧妙地选择分桶字段可以大幅度提升join的性能。通常情况下，分桶字段可以选择经常用在过滤操作或者join操作的字段。

我们可以使用set.hive.enforce.bucketing = true启用分桶设置。

当使用分桶表时，最好将bucketmapjoin标志设置为true，具体配置参数为：

SET hive.optimize.bucketmapjoin = true

CREATE TABLE table_name 
PARTITIONED BY (partition1 data_type, partition2 data_type,….) CLUSTERED BY (column_name1, column_name2, …) 
SORTED BY (column_name [ASC|DESC], …)] 
INTO num_buckets BUCKETS;

### Optimizing Join Operations:

Use map-side joins for small tables that can fit into memory to avoid shuffling data across the network.
Use broadcast joins for joining a small table with a large table, broadcasting the small table to all nodes to avoid data shuffling.
Avoid cross joins (cartesian products) as they can result in a significant increase in data volume and degrade performance.
Optimize join order and join conditions to minimize the amount of data shuffled during join operations.

### Column Pruning:

Avoid using SELECT * and explicitly specify only the columns needed for the query results.
Column pruning reduces the amount of data read from disk and improves query performance.

### Optimizing File Formats:

Choose appropriate file formats such as ORC or Parquet, which are optimized for query performance and storage efficiency.
These file formats support compression and predicate pushdown, which can further improve query performance.
在数据加载过程中，选择合适的数据存储格式（对于结构化数据，可以选择Parquet或ORC等列式存储格式；对于非结构化数据，可以选择TextFile或SequenceFile等格式），可以提高查询性能和减少存储空间。

优化案例

优化前：

LOAD DATA INPATH '/path/to/data' INTO TABLE table;
优化后：

LOAD DATA INPATH '/path/to/data' INTO TABLE table STORED AS ORC;


enable the compression: 
set hive.exec.compress.intermediate=true;
set hive.intermediate.compression.codec=org.apache.hadoop.io.compress.SnappyCodec;
set hive.intermediate.compression.type=BLOCK;
set hive.exec.compress.output=true;

```
org.apache.hadoop.io.compress.DefaultCodec
org.apache.hadoop.io.compress.GzipCodec
org.apache.hadoop.io.compress.BZip2Codec
com.hadoop.compression.lzo.LzopCodec
org.apache.hadoop.io.compress.Lz4Codec
org.apache.hadoop.io.compress.SnappyCodec
```

### Statistics Collection:

Collect table and column statistics using the ANALYZE TABLE command to help the query optimizer make better decisions.
Update statistics regularly, especially after data loading or significant data changes.
Tuning Hive Configuration:

Adjust Hive configuration parameters such as memory allocation, parallelism settings, and query execution parameters based on the characteristics of your workload and cluster resources.
Monitor query performance and resource utilization to identify bottlenecks and fine-tune configuration settings accordingly.

基于成本的优化
Hive在提交最终执行之前会优化每个查询的逻辑和物理执行计划。基于成本的优化会根据查询成本进行进一步的优化，从而可能产生不同的决策：比如如何决定JOIN的顺序，执行哪种类型的JOIN以及并行度等。

可以通过设置以下参数来启用基于成本的优化。

set hive.cbo.enable=true;
set hive.compute.query.using.stats=true;
set hive.stats.fetch.column.stats=true;
set hive.stats.fetch.partition.stats=true;
可以使用统计信息来优化查询以提高性能。基于成本的优化器（CBO）还使用统计信息来比较查询计划并选择最佳计划。通过查看统计信息而不是运行查询，效率会很高。

收集表的列统计信息：

ANALYZE TABLE mytable COMPUTE STATISTICS FOR COLUMNS;
查看my_db数据库中my_table中my_id列的列统计信息：

DESCRIBE FORMATTED my_db.my_table my_id

### optimize the SQL job
使用EXPLAIN命令可以分析查询计划并评估查询的性能。通过查看查询计划中的资源消耗情况，可以找出潜在的性能问题，并进行相应的优化。

优化案例

优化前：

EXPLAIN SELECT * FROM table WHERE age = 30; 
优化后：

EXPLAIN SELECT * FROM table WHERE age = 30 AND partition = 'partition1'; 


根据集群的配置和资源情况，合理调整Hive查询的并行度和资源分配，可以提高查询的并发性和整体性能。通过设置参数hive.exec.parallel值为true，就可以开启并发执行。不过，在共享集群中，需要注意下，如果job中并行阶段增多，那么集群利用率就会增加。建议在数据量大,sql很长的时候使用,数据量小,sql比较的小开启有可能还不如之前快。

SET hive.exec.parallel=true; 
优化后：

SET hive.exec.parallel=false; SET hive.exec.reducers.max=10;


### unbalanced data
在数据仓库中存在大量空值（NULL）的情况下，导致数据分布不均匀的现象。这种数据倾斜可能会对数据分析和计算产生负面影响。当数据仓库中某个字段存在大量空值时，这些空值会在数据计算和聚合操作中引起不平衡的情况。例如，在使用聚合函数（如SUM、COUNT、AVG等）对该字段进行计算时，空值并不会被包括在内，导致计算结果与实际情况不符。数据倾斜会导致部分reduce子任务负载过重，而其他reduce子任务负载较轻，从而影响任务的整体性能。这可能导致任务进度长时间维持在99%（或100%），但仍有少量reduce子任务未完成的情况。
优化方案

第一种：可以直接不让null值参与join操作，即不让null值有shuffle阶段。

第二种：因为null值参与shuffle时的hash结果是一样的，那么我们可以给null值随机赋值，这样它们的hash结果就不一样，就会进到不同的reduce中。

在数据仓库中，不同数据类型的字段可能具有不同的取值范围和分布情况。例如，某个字段可能是枚举类型，只有几个固定的取值；而另一个字段可能是连续型数值，取值范围较大。当进行数据计算和聚合操作时，如果不同数据类型的字段在数据分布上存在明显的差异，就会导致数据倾斜。数据倾斜会导致部分reduce子任务负载过重，而其他reduce子任务负载较轻，从而影响任务的整体性能。这可能导致任务进度长时间维持在99%（或100%），但仍有少量reduce子任务未完成的情况。

优化方案

如果key字段既有string类型也有int类型，默认的hash就都会按int类型来分配，那我们直接把int类型都转为string就好了，这样key字段都为string，hash时就按照string类型分配了。


数据膨胀通常是由于某些数据在仓库中存在大量冗余、重复或者拆分产生的。当这些数据被用于计算和聚合操作时，会导致部分reduce子任务负载过重，而其他reduce子任务负载较轻，从而影响任务的整体性能。

优化方案

在Hive中可以通过参数 hive.new.job.grouping.set.cardinality 配置的方式自动控制作业的拆解，该参数默认值是30。表示针对grouping sets/rollups/cubes这类多维聚合的操作，如果最后拆解的键组合大于该值，会启用新的任务去处理大于该值之外的组合。如果在处理数据时，某个分组聚合的列有较大的倾斜，可以适当调小该值。

在数据仓库中，表连接是常用的操作，用于将不同表中的数据进行关联和合并。然而，当连接键在不同表中的数据分布不均匀时，就会导致连接结果中某些连接键对应的数据量远大于其他连接键的数据量。这会导致部分reduce任务负载过重，而其他任务负载较轻，从而影响任务的整体性能。

优化方案

通常做法是将倾斜的数据存到分布式缓存中，分发到各个Map任务所在节点。在Map阶段完成join操作，即MapJoin，这避免了 Shuffle，从而避免了数据倾斜。


确实无法减少数据量引发的数据倾斜

在某些情况下，数据的数量本身就非常庞大，例如某些业务场景中的大数据集，或者历史数据的积累等。在这种情况下，即使采取了数据预处理、数据分区等措施，也无法减少数据的数量。

优化方案

这类问题最直接的方式就是调整reduce所执行的内存大小。

调整reduce的内存大小使用mapreduce.reduce.memory.mb这个配置。


##### merge small files
在HDFS中，每个小文件对象约占150字节的元数据空间，如果有大量的小文件存在，将会占用大量的内存资源。这将严重限制NameNode节点的内存容量，进而影响整个集群的扩展能力。从Hive的角度来看，小文件会导致产生大量的Map任务，每个Map任务都需要启动一个独立的JVM来执行。这些任务的初始化、启动和执行会消耗大量的计算资源，严重影响性能，因为每个小文件都需要进行一次磁盘IO操作。

--是否和并Map输出文件，默认true

set hive.merge.mapfiles = true;
--是否合并 Reduce 输出文件，默认false

set hive.merge.mapredfiles = true;
--合并文件的大小，默认256000000字节

set hive.merge.size.per.task = 256000000;
--当输出文件的平均大小小于该值时，启动一个独立的map-reduce任务进行文件merge，默认16000000字节

set hive.merge.smallfiles.avgsize = 256000000;

-是否合并小文件，默认true

conf spark.sql.hive.mergeFiles=true;



向量化
Hive中的向量化查询执行大大减少了典型查询操作（如扫描，过滤器，聚合和连接）的CPU使用率。

标准查询执行系统一次处理一行，在处理下一行之前，单行数据会被查询中的所有运算符进行处理，导致CPU使用效率非常低。在向量化查询执行中，数据行被批处理在一起（默认=> 1024行），表示为一组列向量。

要使用向量化查询执行，必须以ORC格式（CDH 5）存储数据，并设置以下变量。

SET hive.vectorized.execution.enabled=true
在CDH 6中默认启用Hive查询向量化，启用查询向量化后，还可以设置其他属性来调整查询向量化的方式，具体可以参考cloudera官网。

谓词下推
默认生成的执行计划会在可见的位置执行过滤器，但在某些情况下，某些过滤器表达式可以被推到更接近首次看到此特定数据的运算符的位置。

比如下面的查询：

select
    a.*,
    b.* 
from 
    a join b on (a.col1 = b.col1)
where a.col1 > 15 and b.col2 > 16
如果没有谓词下推，则在完成JOIN处理之后将执行过滤条件(a.col1> 15和b.col2> 16)。因此，在这种情况下，JOIN将首先发生，并且可能产生更多的行，然后在进行过滤操作。

使用谓词下推，这两个谓词(a.col1> 15和b.col2> 16)将在JOIN之前被处理，因此它可能会从a和b中过滤掉连接中较早处理的大部分数据行，因此，建议启用谓词下推。

通过将hive.optimize.ppd设置为true可以启用谓词下推。

SET hive.optimize.ppd=true

启动严格模式
如果要查询分区的Hive表，但不提供分区谓词（分区列条件），则在这种情况下，将针对该表的所有分区发出查询，这可能会非常耗时且占用资源。因此，我们将下面的属性定义为strict，以指示在分区表上未提供分区谓词的情况下编译器将引发错误。

SET hive.partition.pruning=strict


# avoid data unbalanced.
create table res_tbl as

select n.* from res n

full join org_tbl o on

case when n.id is null then concat('hive', rand()) else n.id end = o.id;

Reducer设置的原则：

每个Reduce处理的数据默认是256MB

hive.exec.reducers.bytes.per.reducer=256000000

每个任务最大的reduce数，默认为1009

hive.exec.reducers.max=1009

计算reduce数的公式

N=min(参数2，总输入数据量/参数1)

设置Reducer的数量

set mapreduce.job.reduces=n

选择使用Tez引擎
Tez: 是基于Hadoop Yarn之上的DAG（有向无环图，Directed Acyclic Graph）计算框架。它把Ｍap/Reduce过程拆分成若干个子过程，同时可以把多个Ｍap/Reduce任务组合成一个较大的DAG任务，减少了Ｍap/Reduce之间的文件存储。同时合理组合其子过程，也可以减少任务的运行时间

设置hive.execution.engine = tez;

通过上述设置，执行的每个HIVE查询都将利用Tez

当然，也可以选择使用spark作为计算引擎

选择使用本地模式
有时候Hive处理的数据量非常小，那么在这种情况下，为查询出发执行任务的时间消耗可能会比实际job的执行时间要长，对于大多数这种情况，hive可以通过本地模式在单节点上处理所有任务，对于小数据量任务可以大大的缩短时间

可以通过

hive.exec.mode.local.auto=true



