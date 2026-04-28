
## 1.Hadoop概述

**核心组件**

**HDFS**（分布式文件系统）：解决海量数据存储

**YARN**（作业调度和集群资源管理的框架）：解决资源任务调度

**MAPREDUCE**（分布式运算编程框架）：解决海量数据计算

**其他框架**

| **框架**  | **用途**                                                  |
| --------- | --------------------------------------------------------- |
| HDFS      | 分布式文件系统                                            |
| MapReduce | 分布式运算程序开发框架                                    |
| ZooKeeper | 分布式协调服务基础组件                                    |
| HIVE      | 基于HADOOP的分布式数据仓库，提供基于SQL的查询数据操作     |
| FLUME     | 日志数据采集框架                                          |
| oozie     | 工作流调度框架                                            |
| Sqoop     | 数据导入导出工具（比如用于mysql和HDFS之间）               |
| Impala    | 基于hive的实时sql查询分析                                 |
| Mahout    | 基于mapreduce/spark/flink等分布式运算框架的机器学习算法库 |

**默认端口更改**

1. 在hadoop3.x之前，多个Hadoop服务的默认端口都属于Linux的临时端口范围（32768-61000）。这就意味着用户的服务在启动的时候可能因为和其他应用程序产生端口冲突而无法启动
2. 现在这些可能会产生冲突的端口已经不再属于临时端口的范围，这些端口的改变会影响NameNode, Secondary NameNode, DataNode以及KMS。与此同时，官方文档也进行了相应的改变，具体可以参见 HDFS-9427以及HADOOP-12811
3. Namenode ports: 50470 --> 9871, 50070--> 9870, 8020 --> 9820
4. Secondary NN ports: 50091 --> 9869,50090 --> 9868
5. Datanode ports: 50020 --> 9867, 50010--> 9866, 50475 --> 9865, 50075 --> 9864
6. Kms server ports: 16000 --> 9600 (原先的16000与HMaster端口冲突)

**YARN** **资源类型**

1. YARN 资源模型（YARN resource model）已被推广为支持用户自定义的可数资源类型（support user-defined countable resource types），不仅仅支持 CPU 和内存；
2. 比如集群管理员可以定义诸如 GPUs、软件许可证（software licenses）或本地附加存储器（locally-attached storage）之类的资源。YARN 任务可以根据这些资源的可用性进行调度；

## 2.Hadoop集群搭建

### 2.1 集群简介

HADOOP集群具体来说包含两个集群：HDFS集群和YARN集群，两者逻辑上分离，但物理上常在一起。

**HDFS集群负责海量数据的存储，**集群中的角色主要有：`NameNode、DataNode、SecondaryNameNode`

**YARN集群负责海量数据运算时的资源调度**，集群中的角色主要有：`ResourceManager、NodeManager`

**mapreduce**是一个分布式运算编程框架，是应用程序开发包，由用户按照编程规范进行程序开发，后打包运行在HDFS集群上，并且受到YARN集群的资源调度管理。

### 2.2 架构模型

**第一种：NameNode与ResourceManager单节点架构模型**

文件系统核心模块：

NameNode：集群当中的主节点，主要用于管理集群当中的各种数据

secondaryNameNode：主要能用于hadoop当中元数据信息的辅助管理

DataNode：集群当中的从节点，主要用于存储集群当中的各种数据

数据计算核心模块：

ResourceManager：接收用户的计算请求任务，并负责集群的资源分配

NodeManager：负责执行主节点APPmaster分配的任务

**第二种：NameNode高可用与ResourceManager单节点架构模型**

文件系统核心模块：

NameNode：集群当中的主节点，主要用于管理集群当中的各种数据，其中NameNode可以有两个，形成高可用状态

DataNode：集群当中的从节点，主要用于存储集群当中的各种数据

JournalNode：文件系统元数据信息管理

数据计算核心模块：

ResourceManager：接收用户的计算请求任务，并负责集群的资源分配，以及计算任务的划分

NodeManager：负责执行主节点ResourceManager分配的任务

**第三种：NameNode单节点与ResourceManager高可用架构模型**

文件系统核心模块：

NameNode：集群当中的主节点，主要用于管理集群当中的各种数据

secondaryNameNode：主要能用于hadoop当中元数据信息的辅助管理

DataNode：集群当中的从节点，主要用于存储集群当中的各种数据

数据计算核心模块：

ResourceManager：接收用户的计算请求任务，并负责集群的资源分配，以及计算任务的划分，通过zookeeper实现ResourceManager的高可用

NodeManager：负责执行主节点ResourceManager分配的任务

**第四种：NameNode与ResourceManager高可用架构模型**

文件系统核心模块：

NameNode：集群当中的主节点，主要用于管理集群当中的各种数据，一般都是使用两个，实现HA高可用

JournalNode：元数据信息管理进程，一般都是奇数个

DataNode：从节点，用于数据的存储

数据计算核心模块：

ResourceManager：Yarn平台的主节点，主要用于接收各种任务，通过两个，构建成高可用

NodeManager：Yarn平台的从节点，主要用于处理ResourceManager分配的任务

### 2.3 集群搭建

https://hadoop.apache.org/ 下载/解压安装包

```xml
vim core-site.xml
--------------------------------
在第19行下添加以下内容：
<!-- 默认文件系统的名称。通过URI中schema区分不同文件系统。-->
<!-- file:///本地文件系统 hdfs:// hadoop分布式文件系统 gfs://。-->
<!-- hdfs文件系统访问地址：http://nn_host:8020。-->
<property>
    <name>fs.defaultFS</name>
    <value>hdfs://node1:8020</value>
</property>
<!-- hadoop本地数据存储目录 format时自动生成 -->
<property>
    <name>hadoop.tmp.dir</name>
    <value>/export/server/hadoop-3.1.4/data</value>
</property>
<!-- 在Web UI访问HDFS使用的用户名。-->
<property>
    <name>hadoop.http.staticuser.user</name>
    <value>root</value>
</property>

```

**配置HDFS路径**

```xml
vim hdfs-site.xml
--------------------------------
在第20行下添加以下内容：
<!-- 设定SNN运行主机和端口。-->
<property>
    <name>dfs.namenode.secondary.http-address</name>
    <value>node2:9868</value>
</property>
```

**配置YARN**

```xml
vim yarn-site.xml
--------------------------------
在第18行下添加以下内容：

<!-- yarn集群主角色RM运行机器。-->
<property>
    <name>yarn.resourcemanager.hostname</name>
    <value>node1</value>
</property>
<!-- NodeManager上运行的附属服务。需配置成mapreduce_shuffle,才可运行MR程序。-->
<property>
    <name>yarn.nodemanager.aux-services</name>
    <value>mapreduce_shuffle</value>
</property>
<!-- 每个容器请求的最小内存资源（以MB为单位）。-->
<property>
  <name>yarn.scheduler.minimum-allocation-mb</name>
  <value>512</value>
</property>
<!-- 每个容器请求的最大内存资源（以MB为单位）。-->
<property>
  <name>yarn.scheduler.maximum-allocation-mb</name>
  <value>2048</value>
</property>
<!-- 容器虚拟内存与物理内存之间的比率。-->
<property>
  <name>yarn.nodemanager.vmem-pmem-ratio</name>
  <value>4</value>
</property>
```

**配置MapReduce**

```xml
vim mapred-site.xml
----------------------------------
在第20行下添加以下内容：
<!-- mr程序默认运行方式。yarn集群模式 local本地模式-->
<property>
  <name>mapreduce.framework.name</name>
  <value>yarn</value>
</property>
<!-- MR App Master环境变量。-->
<property>
  <name>yarn.app.mapreduce.am.env</name>
  <value>HADOOP_MAPRED_HOME=${HADOOP_HOME}</value>
</property>
<!-- MR MapTask环境变量。-->
<property>
  <name>mapreduce.map.env</name>
  <value>HADOOP_MAPRED_HOME=${HADOOP_HOME}</value>
</property>
<!-- MR ReduceTask环境变量。-->
<property>
  <name>mapreduce.reduce.env</name>
  <value>HADOOP_MAPRED_HOME=${HADOOP_HOME}</value>
</property> <property>
  <name>mapreduce.reduce.env</name>
  <value>HADOOP_MAPRED_HOME=${HADOOP_HOME}</value>
</property>
```

**works文件**

```shell
vim /export/server/hadoop-3.1.4/etc/hadoop/workers
----------------------------------
# 删除第一行localhost，然后添加以下三行
node1
node2
node3
```

**修改hadoop.env环境变量**

```shell
hadoop.env文件
vim /export/server/hadoop-3.1.4/etc/hadoop/hadoop-env.sh 
修改第54行为：
export JAVA_HOME=/export/server/jdk1.8.0_241

export HDFS_NAMENODE_USER=root
export HDFS_DATANODE_USER=root
export HDFS_SECONDARYNAMENODE_USER=root
export YARN_RESOURCEMANAGER_USER=root
export YARN_NODEMANAGER_USER=root 
```

**配置环境变量**

```shell
vim /etc/profile
export HADOOP_HOME=/export/server/hadoop-3.1.4
export PATH=$PATH:${HADOOP_HOME}/bin:${HADOOP_HOME}/sbin

source /etc/profile
```

**分发Hadoop安装文件和环境变量**

```shell
cd /export/server/
scp -r hadoop-3.1.4 node2:$PWD
scp -r hadoop-3.1.4 node3:$PWD
scp /etc/profile node2:/etc
scp /etc/profile node3:/etc
在每个节点上执行
source /etc/profile
```

**格式化HDFS**

首次启动HDFS时，必须对其进行格式化操作。本质上是一些清理和准备工作，因为此时的HDFS在物理上还是不存在的。只需要在node1上进行格式化，只能格式化一次

```shell
cd /export/server/hadoop-3.1.4
bin/hdfs namenode -format
```

### 2.4 集群启动

需要启动HDFS和YARN两个集群

```shell
-- 选择node1节点启动NameNode节点
hdfs --daemon start namenode

-- 在所有节点上启动DataNode
hdfs --daemon start datanode

-- 在node2启动Secondary NameNode
hdfs --daemon start secondarynamenode

-- 选择node1节点启动ResourceManager节点
yarn --daemon start resourcemanager

-- 在所有节点上启动NodeManager
yarn --daemon start nodemanager
```

*注意:如果在启动之后，有些服务没有启动成功，则需要查看启动日志，Hadoop的启动日志在每台主机的/export/server/hadoop-x.x.x/logs/目录，需要根据哪台主机的哪个服务启动情况去对应的主机上查看相应的日志，以下是node1主机的日志目录.*

### 2.5 集群关闭

**方式1**

```shell
# 关闭NameNode
hdfs --daemon stop namenode

# 每个节点关闭DataNode
hdfs --daemon stop datanode

# 关闭Secondary NameNode
hdfs --daemon stop secondarynamenode

# 每个节点关闭ResourceManager
yarn --daemon stop resourcemanager

# 每个节点关闭NodeManager
yarn --daemon stop nodemanager
```

**方式2**

```shell
start-dfs.sh
stop-dfs.sh

start-yarn.sh
stop-yarn.sh
```

**方式3**

```shell
-- 一键启动HDFS、YARN
start-all.sh
-- 一键关闭HDFS、YARN
stop-all.sh
```

### 2.6 访问WebUI

NameNode: http://node1:9870

YARN: http://node1:8088

## 3.使用Hadoop

### 3.1 使用HDFS

```shell
#在/export/data/目录中创建a.txt文件，并写入数据
cd /export/data/
touch a.txt
echo "hello" > a.txt 

#将a.txt上传到HDFS的根目录
hadoop fs -put a.txt  /
```

**可以通过http://node1:9870访问HDFS查看是否创建成功**

### 3.2 运行MapReduce程序

> 在Hadoop安装包的share/hadoop/mapreduce下有官方自带的mapreduce程序，可以用以下命令测试

```shell
yarn jar /export/server/hadoop-3.1.4/share/hadoop/mapreduce/hadoop-mapreduce-examples-2.7.5.jar pi 2 50
```

### 3.3 安装目录结构

| bin     | Hadoop最基本的管理脚本和使用脚本的目录，这些脚本是sbin目录下管理脚本的基础实现，用户可以直接使用这些脚本管理和使用Hadoop |
| ------- | ------------------------------------------------------------ |
| etc     | Hadoop配置文件所在的目录，包括core-site,xml、hdfs-site.xml、mapred-site.xml等从Hadoop1.0继承而来的配置文件和yarn-site.xml等Hadoop2.0新增的配置文件。 |
| include | 对外提供的编程库头文件（具体动态库和静态库在lib目录中），这些头文件均是用C++定义的，通常用于C++程序访问HDFS或者编写MapReduce程序。 |
| lib     | 该目录包含了Hadoop对外提供的编程动态库和静态库，与include目录中的头文件结合使用。 |
| libexec | 各个服务对用的shell配置文件所在的目录，可用于配置日志输出、启动参数（比如JVM参数）等基本信息。 |
| sbin    | Hadoop管理脚本所在的目录，主要包含HDFS和YARN中各类服务的启动/关闭脚本。 |
| share   | Hadoop各个模块编译后的jar包所在的目录，官方自带示例。        |

**配置文件：**

##### hadoop-env.sh

文件中设置的是Hadoop运行时需要的环境变量。JAVA_HOME是必须设置的，即使我们当前的系统中设置了JAVA_HOME，它也是不认识的，因为Hadoop即使是在本机上执行，它也是把当前的执行环境当成远程服务器。

##### core-site.xml

hadoop的核心配置文件，有默认的配置项core-default.xml

core-default.xml与core-site.xml的功能是一样的，如果在core-site.xml里没有配置的属性，则会自动会获取core-default.xml里的相同属性的值。

##### hdfs-site.xml

HDFS的核心配置文件，主要配置HDFS相关参数，有默认的配置项hdfs-default.xml

hdfs-default.xml与hdfs-site.xml的功能是一样的，如果在hdfs-site.xml里没有配置的属性，则会自动会获取hdfs-default.xml里的相同属性的值。

##### mapred-site.xml

MapReduce的核心配置文件，Hadoop默认只有个模板文件mapred-site.xml.template,需要使用该文件复制出来一份mapred-site.xml文件

##### yarn-site.xml

YARN的核心配置文件,在该文件中的<configuration>标签中添加以下配置

##### workers

workers文件里面记录的是集群主机名，配合一键启动脚本如start-dfs.sh、stop-yarn.sh用来进行集群启动。这时候workers文件里面的主机标记的就是从节点角色所在的机器

### 3.4 测试读写速度

1. 启动集群

2. 启动写入基准测试

   ```shell
   hadoop jar hadoop-3.1.4/share/hadoop/mapreduce/hadoop-mapreduce-client-jobclient-3.1.4-tests.jar  TestDFSIO -write -nrFiles 10  -fileSize 10MB
   
   2023-12-18 15:51:59,799 INFO fs.TestDFSIO: ----- TestDFSIO ----- : write
   2023-12-18 15:51:59,800 INFO fs.TestDFSIO:             Date & time: Mon Dec 18 15:51:59 CST 2023
   2023-12-18 15:51:59,800 INFO fs.TestDFSIO:         Number of files: 10
   2023-12-18 15:51:59,800 INFO fs.TestDFSIO:  Total MBytes processed: 50
   2023-12-18 15:51:59,801 INFO fs.TestDFSIO:       Throughput mb/sec: 15.09
   2023-12-18 15:51:59,801 INFO fs.TestDFSIO:  Average IO rate mb/sec: 19.14
   2023-12-18 15:51:59,801 INFO fs.TestDFSIO:   IO rate std deviation: 9.13
   2023-12-18 15:51:59,801 INFO fs.TestDFSIO:      Test exec time sec: 47.94
   ```

3. 启动读取基准测试

   ```shell
   hadoop jar share/hadoop/mapreduce/hadoop-mapreduce-client-jobclient-3.3.6-tests.jar TestDFSIO -read -nrFiles 10 -fileSize 5MB
   
   2023-12-18 15:54:15,511 INFO fs.TestDFSIO: ----- TestDFSIO ----- : read
   2023-12-18 15:54:15,511 INFO fs.TestDFSIO:             Date & time: Mon Dec 18 15:54:15 CST 2023
   2023-12-18 15:54:15,511 INFO fs.TestDFSIO:         Number of files: 10
   2023-12-18 15:54:15,511 INFO fs.TestDFSIO:  Total MBytes processed: 50
   2023-12-18 15:54:15,511 INFO fs.TestDFSIO:       Throughput mb/sec: 74.29
   2023-12-18 15:54:15,511 INFO fs.TestDFSIO:  Average IO rate mb/sec: 101.05
   2023-12-18 15:54:15,511 INFO fs.TestDFSIO:   IO rate std deviation: 55.18
   2023-12-18 15:54:15,511 INFO fs.TestDFSIO:      Test exec time sec: 65.45
   2023-12-18 15:54:15,511 INFO fs.TestDFSIO: 
   ```

4. 清理测试数据`hadoop jar share/hadoop/mapreduce/hadoop-mapreduce-client-jobclient-3.3.6-tests.jar TestDFSIO -clean`

# HDFS

## 1.目录规划

| 目录       | 说明                                               |
| ---------- | -------------------------------------------------- |
| /source    | 用于存储原始采集数据                               |
| /common    | 用于存储公共数据集，例如：IP库、省份信息、经纬度等 |
| /workspace | 工作空间，存储各团队计算出来的结果数据             |
| /tmp       | 存储临时数据，每周清理一次                         |
| /warehouse | 存储hive数据仓库中的数据                           |

## 2.指令操作

Hadoop提供了文件系统的shell命令行客户端，使用方法： `hdfs [SHELL_OPTIONS] COMMAND  [GENERIC_OPTIONS] [COMMAND_OPTIONS]`  

选项：

| **COMMAND_OPTIONS**     | **Description**                                     |
| ----------------------- | --------------------------------------------------- |
| SHELL_OPTIONS           | 常见的shell选项，例如：操作文件系统、管理hdfs集群.. |
| GENERIC_OPTIONS         | 多个命令支持的公共选项                              |
| COMMAND COMMAND_OPTIONS | 用户命令、或者是管理命令                            |

示例：

1. 查看HDFS中/parent/child目录下的文件或者文件夹 `hdfs  dfs -ls  /parent/child`

2. 所有HDFS命令都可以通过bin/hdfs脚本执行；

3. 还有一个hadoop命令也可以执行文件系统操作，还可以用来提交作业，此处我们均使用hdfs，为了更好地区分和对hdfs更好的支持；

4. mkdir命令

   ```shell
   # 以<paths>中的URI作为参数，创建目录。使用-p参数可以递归创建目录
   hdfs dfs -mkdir /common
   hdfs dfs -mkdir /workspace
   hdfs dfs -mkdir /tmp/
   hdfs dfs -mkdir /warehouse
   hdfs dfs -mkdir /source
   ```

5. ls命令：`hdfs dfs -ls /`，-R表示递归展示目录下的内容

6. put命令：`hdfs dfs -put ...`

7. rm命令：`hdfs dfs -rm ...`，删除参数指定的文件和目录，参数可以有多个，那么在回收站可用的情况下，该选项将跳过回收站而直接删除文件；否则，在回收站可用时，执行命令后会将文件暂时放到回收站中；

8. moveFromLocal命令： `hdfs dfs -moveFromLocal <localsrc> <dst> `

9. 查看hdfs文件的内容：

   1. get命令，从HDFS下载文件到Linux；将文件拷贝到本地文件系统，可以通过指定-ignorecrc选项拷贝CRC校验失败的文件。-crc选项表示获取文件以及CRC校验文件；`hdfs dfs -get [-ignorecrc ] [-crc] <src> <localdst>`
   2. less命令，在Linux上查看下载的文件；
   3. head命令：显示要输出的文件的开头的1KB数据；`hdfs dfs -head ... `
   4. tail命令：显示文件结尾的1KB数据，与Linux一样-f选项表示数据只要有变化也会输出到控制台；`hdfs dfs -tail [-f] ...`

10. cp命令：将文件拷贝到目标路径中，如果<dest>为目录的话，可以将多个文件拷贝到该目录下，-f选项将覆盖目标，如果它已经存在，-p选项将保留文件属性；`hdfs dfs -cp ... <dest>`

11. appendToFile命令：追加一个或多个文件到hdfs指定文件中，也可以从命令行读取输入；`hdfs dfs -appendToFile <localsrc> ... <dest>`

12. df命令：`hdfs dfs -df -h /`

13. du命令：`hdfs dfs -du`

    1. -s：表示显示文件长度的汇总摘要，而不是单个文件的摘要；
    2. -h：选项将以“人类可读”的方式格式化文件大小；
    3. -v：选项将列名显示为标题行；
    4. -x：选项将从结果计算中排除快照；

14. mv命令：从原路径移动到目标路径，该命令不能跨文件系统；`hdfs dfs -mv ... <dest>`

15. **setrep命令**：减少副本数提升存储资源利用率，-w标志请求命令等待复制完成，可能会耗费很长时间，-R是为了向后兼容，没有作用；`hdfs dfs -setrep [-R] [-w] <numReplicas> <path>`








