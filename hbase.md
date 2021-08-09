# HBase简介

HBase定义：HBase是一种分布式、可扩展、支持海量数据存储的NoSQL数据库。

## 数据模型

逻辑上，HBase的数据模型同关系型数据库很类似，数据存储在一张表中，有行有列。但从HBase的底层物理存储结构（K-V）来看，HBase更像一个multi-dimensional map。

## 逻辑结构

列族、列、row key、Region

![image-20210730075723000](C:\Users\CW\Desktop\笔记\计算机软件\HBase\pic\image-20210730075723000.png)

Region是一张表的横向切片。

row key按字典排序。

## 物理结构

![image-20210730080706540](C:\Users\CW\Desktop\笔记\计算机软件\HBase\pic\image-20210730080706540.png)

TimeStamp 不同版本的数据根据timestamp进行区分。

## 数据模型

1. ### 命名空间（Name Space）

   类似于关系型数据库的database概念，每个命名空间下有多个表。HBase有两个自带的命名空间，分别是hbase和default。hbase中存放的是HBase的内置表，default表是用户默认使用的命名空间。

2. ### Region

   HBase定义表的时候只需要声明列族即可，不需要声明具体的列。这意味着，往HBase写入数据时，字段可以动态、按需执行。HBase可以轻松应对字段变更的场景。

   建表的时候不需要定义列，只需要增加列族。列可以动态增加。

3. ### Row

   每一行数据由一个RowKey和多个Column组成，数据是按照RowKey的字典顺序存储的，并且查询数据时只能根据RowKey进行检索，所以RowKey‘的设计十分重要。

4. ### Column

   HBase中的没个列都由Column Family（列族）和Column Qualifier（列限定符）进行限定。info:name

5. ### TimeStamp

   用于标识数据的不同版本，没条数据写入时，如果不指定时间戳。系统会自动加入该字段。

6. ### Cell

   由{rowkey、column family、col-qualifier、timestamp}组成。cell没有数据类型，全是字节码形式存储。

## HBase基本架构

![image-20210730083450584](C:\Users\CW\AppData\Roaming\Typora\typora-user-images\image-20210730083450584.png)

# 快速入门

## 安装

1. 修改regionservers配置文件

   ```properties
   hadoop102
   hadoop103
   hadoop104
   ```

2. hbase-env.sh

   ```sh
   export JAVA_HOME = 
   export HBASE_MANAGES_ZK = false
   ```

3.  hbase-site.xml

   ```xml
   <property>
       <name>hbase.rootdir</name>
       <value>hdfs://ns1/HBase</value>
   </property>
   <property>
       <name>hbase.cluster.distributed</name>
       <value>true</value>
   </property>
   <property>
       <name>hhbase.master.port</name>
       <value>16000</value>
   </property>
   <property>
       <name>hbase.zookeeper.quorum</name>
       <value>hadoop102,hadoop103,hadoop104</value>
   </property>
   <property>
       <name>hbase.zookeeper.property.dataDir</name>
       <value>/opt/module/zookeeper-3.4.10/zkData</value>
   </property>
   ```

4. 软连接hdfs-site.xml、core-site.xml到conf目录。

5. 启动集群

   ```sh
   bin/hbase-deamon.sh start master
   bin/hbase-deamon.sh start regionserver
   ```

6. shell操作

   ```shell
   hbase shell
   ```

7.  hbase shell 命令

   ```sql
   help
   list 查询用户表
   create 'stu', 'info'
   create 'student', 'info', 'info2'
   describe 'stu'
   alter 'stu', {NAME=>'info', VERSION=>3}
   disable 'stu'
   drop 'stu'
   
   list_namespace
   create_namespace 'bigdata'
   create 'bigdata:stu', 'info'
   
   put 'stu','1001','info:name','zhangsan'
   scan 'stu'
   get 'stu', '1001'
   get 'stu', '1001', 'info:name'
   
   put 'stu','1001','info:name','zhangsansan'
   scan 'stu', {RAW=>true, VERSION=>10} --获取10个版本的数据。
   delete 'stu','1001','info:sex'
   
   delete 'stu','1001','info' --API可删，命令行失败
   deleteall 'stu', '1001'
   
   get 'stu', '1005', {COLUMN=>'info1:name', VERSION=>3}
   ```

   数据存储在hdfs://hadoop102:9000/HBase/data/namespace/table/region/columnfamily


# 详细架构

![image-20210801131850792](C:\Users\CW\Desktop\笔记\计算机软件\HBase\pic\image-20210801131850792.png)

## 写流出

HBase读比写慢。

![image-20210801132708196](C:\Users\CW\Desktop\笔记\计算机软件\HBase\pic\image-20210801132708196.png)

注意：写wal的时候没有做同步，先写MemStore，写完MemStore才会做wal同步。如果同步失败，则会回滚。

源码：HBaseRegion STEP1-9

## Flush流程

![image-20210801140102802](C:\Users\CW\Desktop\笔记\计算机软件\HBase\pic\image-20210801140102802.png)

```xml
<property>
    <name>hbase.regionserver.global.memstore.size</name>
    <value></value>
    <description>
        Maximum size of all memstores in a region server before new updates are blocked and flushes are forced. Defaults to 40% if heap(0.4). Updates are blocked and flushes are forced until size of all memstores in a region server hits hbase.regionserver.global.memstore.size.lower.limit. The default value in this configuration has been intentionally left empty in order to honor the old hbase.regionserver.global.memstore.upperLimit property if present.
        触发阻塞写
    </description>
</property>
<property>
    <name>hbase.regionserver.global.memstore.size.lower.limit</name>
    <value></value>
    <description>
        Maximum size of all memstores in a region server before flushes are forced. Defaults to 95% of hbase.regionserver.global.memstore.size. A 100% value for this value cause the minimum possible flushing to occur when updates are blocked due to memstore limiting.
    </description>
</property>
<property>
    <name>hbase.regionserver.optionalcacheflushinterval</name>
    <value>3600000</value>
    <description>
        Maximum amount of time an edit lives in memory before being automically flushed.
        Default 1 hour.
        最后一次编辑时间
    </description>
</property>
<property>
    <name>hbase.hregion.memstore.flush.size</name>
    <value>134217728</value>  128M
    <description>
		Memstore will flushed to disk if size of the memstore exceeds this number of bytes. Value is checked by a thread that runs every hbase.server.thread.wakefrequency.
    </description>
</property>
```

## 读流程

![image-20210801143147823](C:\Users\CW\Desktop\笔记\计算机软件\HBase\pic\image-20210801143147823.png)

读数据的时候，会读StoreFile写到BlockedCahe，然后与Memstore的数据作比较，取时间戳比较大的记录。

## StoreFile Compaction

由于memstore每次刷写都会生成一个新的HFile，且同一个字段的不同版本和不同类型有可能会分布在不同的Hfile中，因此查询时需要遍历所有的HFile。为了减少HFile的数量以及清理掉过期的数据，会进行StoreFile Compaction。

Compaction分为两种，分别是Minor Compaction和Major Compaction。Minor Compaction会将临近的若干个较小的HFile合并成一个较大的HFile，但不会清理过期和删除的数据。Major Compaction会将一个Store下的所有HFile合并成一个大的HFile，并且会清理掉过和删除的数据。

![image-20210801145858267](C:\Users\CW\Desktop\笔记\计算机软件\HBase\pic\image-20210801145858267.png)

```xml
<property>
    <name>hbase.hregion.majorcompaction</name>
    <value>604800000</value>
    <description>
        隔7天进行major合并一次，建议关闭。
    </description>
</property>
<property>
    <name>hbase.hstore.compactionThreshold</name>
    <value>3</value>
    <description>
        HStoreFile的文件数超过配置值，会进行合并。>=3
    </description>
</property>
```

## 数据真正删除的时间

flush删除数据的条件：数据都在memstore，则会删除memstore中的相同rowkey的数据。

major compact也会删除数据

## Region Split

```properties
hbase.hregion.max.filesize
```

当一个region中的某个Store下所有StoreFile的总大小超过Min(R^2 * hbase.hregion.memstore.flush.size, hbase.hregion.max.filesize)，该region就会进行拆分，其中R是当前RegionServer中属于该Table的个数。

自动切分会产生数据倾斜。

建表时要进行域分区