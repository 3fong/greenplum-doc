# greenplum

数据分析技术趋势主要包括：

云原生分布式：无论是OLTP还是OLAP，如今单机数据已无法满足企业业务和数据快速增长的需求，分布式数据库成为主流，同时数据库市场未来主要在云上（"Gartner: The Future of the DBMS Market Is Cloud")，云原生架构与特性成为云数据库的必要条件。    
存储计算分离：云计算的本质是资源高效池化，而数据库的两大核心组件是存储和计算。通过存储计算分离，做到两者解耦，资源池化，独立扩展，满足业务上资源隔离，数据共享的需求，是当下的架构趋势。    
计算分析一体化：传统数据分析方案是定期从OLTP系统抽取数据同步到OLAP系统，有些可以做到准实时同步。该方案带来的问题是部署复杂，实时性低，数据冗余和高成本。理想情况是一套HTAP系统同时提供计算和分析。    
大数据与数据库一体化：早期大数据技术以牺牲一定程度一致性为基础提供分布式能力，解决了传统单机数据库的扩展性不足问题，在MapReduce基础上提供了标准SQL接口，架构上也逐步采用了部分MPP数据库技术；另一方面，分布式数据库也快速发展，融合了部分大数据技术和存储格式，在扩展性层面获得了很好提升。在数据分析场景，两者解决的都是相同问题。    



- 适合场景


业务逻辑极其复杂:如金融行业.论业务逻辑简单还是复杂，OLTP 还是 OLAP 负载，PostgreSQL 都可以支持     
PostgreSQL 不止是一个数据库，更是一个强大的开发平台:强大的数据库内部 function 支持，而且可以用多种语言编写，对于复杂业务逻辑计算以及大数据量访问完全可以在数据库本地化实现，大大减少了网络交互成本，从而整体提升应用性能。    
商业开源    

mysql的问题:

```
就是数据库只做为数据的存储，只提供简单的查询访问。而复杂的业务逻辑前移到应用服务器端来完成。MySQL 数据库自身的特性并不十分丰富，例如 innoDB 存储引擎只提供索引组织表形式的数据存储格式，某种程度上限制了它的使用场景。对于触发器和存储过程的支持较弱，并不建议使用。应用的 CRUD 操作尽量通过主键进行，虽然支持二级索引，但通过二级索引操作会有性能损失。在进行关系型数据库中必不可少的表关联操作时，只支持 Nested Loops 关联方法，缺少对 sort merge join 和 hash join 的支持。当关联表超过 2 张时，MySQL 的优化器有时生成的执行计划不优，造成性能下降
```


## 架构

Greenplum主要由Master节点、Segment节点、interconnect三大部分组成。Greenplum master是Greenplum数据库系统的入口，接受客户端连接及提交的SQL语句，将工作负载分发给其它数据库实例（segment实例），由它们存储和处理数据。Greenplum interconnect负责不同PostgreSQL实例之间的通信。Greenplum segment是独立的PostgreSQL数据库，每个segment存储一部分数据。大部分查询处理都由segment完成。

Master节点不存放任何用户数据，只是对客户端进行访问控制和存储表分布逻辑的元数据
Segment节点负责数据的存储，可以对分布键进行优化以充分利用Segment节点的io性能来扩展整集群的io性能
存储方式可以根据数据热度或者访问模式的不同而使用不同的存储方式。一张表的不同数据可以使用不同的物理存储方式：行存储、列存储、外部表

### 实现

1. 分片存储

在Greenplum中每个表都是分布在所有节点上的。Master节点首先通过对表的某个或多个列进行hash运算，然后根据hash结果将表的数据分布到Segment节点中。整个过程中Master节点不存放任何用户数据，只是对客户端进行访问控制和存储表分布逻辑的元数据。

![Greenplum存储结构](https://img-blog.csdn.net/20180108151432230?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvZGNwa2VrZQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

2. 多态存储

Greenplum提供称为“多态存储”的灵活存储方式。多态存储可以根据数据热度或者访问模式的不同而使用不同的存储方式。一张表的不同数据可以使用不同的物理存储方式。支持的存储方式包含：

行存储：行存储是传统数据库常用的存储方式，特点是访问比较快，多列更新比较容易。    
列存储：列存储按列保存，不同列的数据存储在不同的地方（通常是不同文件中）。适合一次只访问宽表中某几个字段的情况。列存储的另外一个优势是压缩比高。    
外部表：数据保存在其他系统中例如HDFS，数据库只保留元数据信息。    

3. 并行查询计划和执行

下图为一个简单SQL语句，从两张表中找到2008年的销售数据。图中右边是这个SQL的查询计划。从生成的查询计划树中看到有三种不同的颜色，颜色相同表示做同一件事情，我们称之为分片/切片（Slice）。最下层的橙色切片中有一个重分发节点，这个节点将本节点的数据重新分发到其他节点上。中间绿色切片表示分布式数据关联（HashJoin）。最上面切片负责将各个数据节点收到的数据进行汇总。

![](https://img-blog.csdn.net/20180108151534700?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvZGNwa2VrZQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

然后看看这个查询计划的执行。主节点（Master）上的调度器（QD）会下发查询任务到每个数据节点，数据节点收到任务后（查询计划树），创建工作进程（QE）执行任务。如果需要跨节点数据交换（例如上面的HashJoin），则数据节点上会创建多个工作进程协调执行任务。不同节点上执行同一任务（查询计划中的切片）的进程组成一个团伙（Gang）。数据从下往上流动，最终Master返回给客户端

![](https://img-blog.csdn.net/20180108151550687?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvZGNwa2VrZQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

4. 并行数据加载

（1）并行加载技术充分利用分布式计算和分布式存储的优势，保证发挥出每一块Disk的I/O资源    
（2）并行加载比串行加载，速度提高40-50倍以上，减少ETL窗口时间    
（3）增加Segment和ETL Server，并行加载速度呈线性增长    

5. sql语言.使用简单

6. 生态好.支持主流应用集成

### 核心技术

1 mpp

2 mvcc

3 multi-master

分布式事务    
事务共享能力    
容错和高可用    


### 外部表

[](https://www.cnblogs.com/xibuhaohao/p/11127735.html)

https://blog.csdn.net/MyySophia/article/details/101693314

### 参考资料

[云原生数据仓库AnalyticDB PostgreSQL版 白皮书](https://help.aliyun.com/document_detail/211056.html)






