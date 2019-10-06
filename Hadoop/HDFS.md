# HDFS

[TOC]

------

## RAID 时代的数据存储解决方案

单机时代：采用单块磁盘进行数据存储和读写的方式，由于寻址和读写的时间消耗，导致I/O性能非常低，且存储容量还会受到限制。

<u>缺点：单块磁盘极其容易出现物理故障，经常导致数据的丢失</u>

RAID （ Redundant Array of Independent Disks ）即独立磁盘冗余阵列，简称为「磁盘阵列」，其实就是用多个独立的磁盘组成在一起形成一个大的磁盘系统，从而实现比单块磁盘更好的存储性能和更高的可靠性。

常用RAID：

1. **RAID0**  将多块硬盘串联

     优点：读写性能非常高

     缺点：一个硬盘坏了就全坏了

2. **RAID1**  将多块硬盘并联

    优点：安全性很高，数据都备份

    缺点：容量不会增加

3. **RAID10**  先将硬盘数量/2，组RAID1；每组RAID串联，组成大RAID0

    缺点：需要更多的硬盘组合

## HDFS的简介：

Hadoop Distributed File System -- HDFS

HDFS

HDFS数据块，一般默认为128MB。磁盘的块大小一般为512字节。

<img src="F:\hadoop\picture\1569590738152.png" alt="1569590738152" style="zoom: 67%;" />



1、文件有大小，200M-100T，充分利用分布式文件系统特性，将不管多大的文件都按固定大小分成HDFS数据块

2、数据块的安全性：HDFS有数据块的副本机制，保证一个数据块都不会丢失

3、HDFS有统一的读写机制：NameNode（具备HA机制）



## HDFS的基本思想

大数据时代，1、需要将数据集进行分区并存储到若干台独立计算机

​                        2、需要一种文件系统管理多台机器的文件（分布式文件系统）                        

<img src="F:\hadoop\picture\0.7621709530514569.png" alt="img" style="zoom:50%;" />

## HDFS体系结构中角色的划分

![1570188977460](F:\hadoop\picture\1570188977460.png)

###### 	NameNode：

- 维护和管理DataNodes
- 管理文件系统namespace并控制client对应的访问权限
- 记录所有存储在集群中的文件的元信息。eg: blocks存储的位置、文件的大小、权限、文件结构等，有两个文件和元数据关联着
  `FsImage`
  保存了最新的元数据检查点，包含了整个HDFS文件系统的所有目录和文件的信息。对于文件来说包括了数据块描述信息、修改时间、访问时间等；对于目录来说包括修改时间、访问权限控制信息(目录所属用户，所在组)等。
  一般开始时namenode的操作都放在EditLog中，然后通过异步更新。
  `EditLog`
  记录最近通过namenode对文件系统的所有修改操作。
- 记录文件系统的所有操作元数据。 存储在 EditLogs
- 维护着与DataNodes的心跳检测
- DataNodes磁盘存储均衡、DataNodes故障转移

###### DataNode

数据存储节点

###### Secondary NameNode



![img](https://upload-images.jianshu.io/upload_images/3359971-2815d2554c48ba8e.png?imageMogr2/auto-orient/strip|imageView2/2/w/528/format/webp)

它的主要职责

- 备用节点，也称为standby namenode。NameNode是HDFS的大脑核心，一旦NameNode出现不可用，那么整个HDFS集群将不可用，Secondary NameNode作为NameNode的备用节点，进行NameNode容错
- 负责合并Editlogs和FsImage
- 定时从 namenode 下载Editlogs并和现有FsImage进行合并，然后将合并后的FsImage更新到namenode

*FailoverController*
故障切换器，管理着将活动namenode转移为备用namenode的过程，默认通过ZK来确保仅有一个活跃namenode。每一个namenode都有一个运行着的故障转移器。

###### Balancer

用于平衡DataNode集群之间各节点的磁盘利用率。						



## ZKFC详解

> ZKFailoverController(ZKFC)是一个新的组件，它是一个ZooKeeper客户端，它还监视和管理NameNode的状态。**运行NameNode的每台机器也运行ZKFC**，他们之间是一对一的关系。
>
> ZKFC负责：
>   
>- **健康监测**-
> ZKFC定期使用健康检查命令调用其本地NameNode。只要NameNode以健康的状态及时响应，ZKFC就会认为节点是健康的。
>   如果节点已崩溃、冻结或以其他方式进入不健康状态，则健康监视器将将其标记为不健康。
>   - **ZooKeeper会话管理**
> 当本地NameNode健康时，ZKFC在ZooKeeper中举行一个开放的会话。
>   如果本地NameNode是活动的，它也持有一个特殊的“锁”。此锁使用ZooKeeptor对“临时”节点的支持；如果会话过期，则将自动删除锁节点。
>   
>- **基于ZooKeeper的选举**
> 如果本地NameNode是健康的，而ZKFC认为目前没有其他节点持有锁，
>   它本身就会尝试获取锁。如果它成功了，那么它已经“赢得了选举”，并负责运行故障转移以使其本地NameNode活动。故障转移过程类似于上面描述的手动故障转移：首先，如果需要，对前一个活动进行隔离，然后本地NameNode转换到活动状态。

## HDFS数据的复制

## HDFS读取数据的流程

1. 客户端调用DistributedFileSystem 的 Open() 方法打开文件。
2. DistributedFileSystem 用 **RPC** 连接到 NameNode，请求获取文件的数据块的信息；NameNode 返回文件的部分或者全部数据块列表；对于每个数据块，NameNode 都会返回该数据块副本的 DataNode 地址；DistributedFileSystem 返回 FSDataInputStream 给客户端，用来读取数据。
3. 客户端调用 FSDataInputStream 的 Read() 方法开始读取数据。
4. FSInputStream 连接保存此文件第一个数据块的最近的 DataNode，并以数据流的形式读取数据；客户端多次调用 Read()，直到到达数据块结束位置。
5. FSInputStream连接保存此文件下一个数据块的最近的 DataNode，并读取数据。
6. 当客户端读取完所有数据块的数据后，调用 FSDataInputStream 的 Close() 方法。


## HDFS写入数据的流程
 1.客户端调用 DistribuedFileSystem 的 Create() 方法来创建文件。
         2.DistributedFileSystem 用 RPC 连接 NameNode，请求在文件系统的命名空间中创建一个新的文件；NameNode 首先确定文件原来不存在，并且客户端有创建文件的权限，然后创建新文件；DistributedFileSystem 返回 FSOutputStream 给客户端用于写数据。
         3.客户端调用 FSOutputStream 的 Write() 函数，向对应的文件写入数据。
         4.当客户端开始写入文件时，FSOutputStream 会将文件切分成多个分包（Packet），并写入其內部的数据队列。FSOutputStream 向 NameNode 申请用来保存文件和副本数据块的若干个 DataNode，这些 DataNode 形成一个数据流管道。
队列中的分包被打包成数据包，发往数据流管道中的第一个 DataNode。第一个 DataNode 将数据包发送给第二个 DataNode，第二个 DataNode 将数据包发送到第三个 DataNode。这样，数据包会流经管道上的各个 DataNode。

5. 为了保证所有 DataNode 的数据都是准确的，接收到数据的 DataNode 要向发送者发送确认包（ACK Packet）。确认包沿着数据流管道反向而上，从数据流管道依次经过各个 DataNode，并最终发往客户端。当客户端收到应答时，它将对应的分包从内部队列中移除。
6. 不断执行第 (3)~(5)步，直到数据全部写完。
7. 调用 FSOutputStream 的 Close() 方法，将所有的数据块写入数据流管道中的数据结点，并等待确认返回成功。最后通过 NameNode 完成写入。 





## HDFS的安装和配置

![1570197881614](F:\hadoop\picture\1570197881614.png)

## HDFS常用操作介绍
 1. hadoop fs -ls  展示HDFS某个目录下的内容 （-h：文件大小显示为最大单位）（-R：递归的角度显示）

 2. 上传  hadoop fs -put  需要上传的文件 /data（默认hdfs目录下） （-f:强制上传）

      1. 创建目录并且上传文件  hadoop  fs -mkdir  /ns2/usr/su/d1  && hadoop fs -put  ./file* /ns2/usr/su/d1/

 3. 下载  hadoop fs -get 需要下载的文件  hadoop fs -get hdfs:/test.txt /home/hadoop/

 4. 拷贝从本地到HDFS      hadoop fs -get hdfs:/test.txt /home/hadoop/

 5. **拷贝文件并重命名** hadoop fs -get /test.txt ~/hadoop/test.txt

 6. 移动  hadoop fs -mv /ns2/usr/file1  /ns2/usr/su/

 7. 删除文件    hadoop fs -rm /a.txt  

 8.  hdfs不推荐写法 但是也能对付删(不推荐)    hadoop fs -rmr /dir/          hadoop fs -rm -r /dir/

 9. 读取文件  hadoop fs -cat /test.txt

 10. 查看尾部1k字节 hadoop fs -tail /test.txt 

 11. 查看空文件  hadoop fs - touchz /newfile.txt

 12. 读取本地文件内容并追加到HDFS hadoop fs -appendToFile file:/test.txt hdfs:/newfile.txt 

 13. 改变文件副本数   hadoop fs -setrep -R -w 2 /test.txt

 14. 获取HDFS目录的物理空间信息 hadoop fs -count   

     第一个数值表示/下的目录的个数（包括其本身）；

     第二个数值表是当前目录下文件的个数；

     第三个数值表示该目录下文件所占的空间大小，这个大小是不计算副本的个数的。
     
     
     
 15. 管理工具 ：

         **-report：*  查看文件系统的基本信息和统计信息。 hdfs dfsadmin -report

          *-safemode ：**安全模式命令。安全模式下，你不能向HDFS写入数据，但是你可以读出数据

          hdfs dfsadmin -safemode enter

          hdfs dfsadmin -safemode leave

       ​    hdfs dfsadmin -safemode get

        安全模式是NameNode的一种状态，在这种状态下，NameNode不接受对名字空间的更改（只读）；不复制或删除块。NameNode在启动时自动进入安全模式，当配置块的最小百分数满足最小副本数的条件时，会自动离开安全模式。enter是进入，leave是离开。get是获取安全模式信息

       注意：一般一个HDFS集群-》正常操作的时候 是只有刚开始一会儿功夫进入到安全模式，然后很快就会推出 正常使用；

 16. 如何新增一个datenode

       1. 硬盘对刻 ：新电脑系统正常启动 ；虚拟机：s3克隆一份s4

       2. 对新机器进行修改 hosts、ips文件、slaves文件

       3. 新机器上启动datenode

          

        

       

       

​     

​     

​     

​     


## 疑问

为什么HDFS不接受小文件的存储？ --占用内存过大

## 参考链接

图文详解 https://www.jianshu.com/p/8b4dc5a154e5

ZKFC []https://www.jianshu.com/p/db6a92ecd791



