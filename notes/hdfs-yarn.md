## HDFS, Yarn关键技术点整理

> 本文内容由 https://github.com/666666666666 整理 

### Hadoop

* HDFS架构简介：https://hadoop.apache.org/docs/r2.7.0/hadoop-project-dist/hadoop-hdfs/HdfsDesign.html
 
* HDFS各个角色的作用：
  **NameNode**
维护着文件系统树(filesystem tree)以及文件 树中所有inode的(文件和文件夹)的元数据(metadata)。对于文件来说包括了数据块描述信息、修改时间、访问时间等；对于目录来说包括修改时间、访问权限控制信息(目录所属用户，所在组)等。
 **DataNode**
管理本地存储，响应客户端的读写请求，在namenode的指导下进行块的创建，删除，复制。
 **ZKFC(可选)**
当需要NameNode HA自动Failover时必选。用于管理NameNode状态。主要包括以下三个功能：Health monitoring，ZooKeeper session management，ZooKeeper-based election
详细介绍：https://hadoop.apache.org/docs/r2.7.0/hadoop-project-dist/hadoop-hdfs/HDFSHighAvailabilityWithQJM.html#Automatic_Failover
 **JournalNode(可选)**
 当需要NameNode HA时必选。两个NameNode为了保持数据一致需要通过JournalNode通信。最多允许 (N - 1) / 2个节点失败，所以建议部署奇数个实例。Active的namenode修改元数据时会持久化一条记录到JournalNodes的master节点。Standby的Namenode会监听这些变化并应用到自己的元数据上。
 
* HDFS + yarn 部署流程
 (1) 修改主机名
 (2) 配置SSH免密码登录
 (3) 安装JRE
 (4) 下载Hadoop发布版，解压缩到服务器
 (5) 配置core-site.xml，hdfs-site.xml,slaves
 (6) NameNode格式化
 (7) 启动 ./sbin/start-dfs.sh
 (8) 配置mapred-site.xml，yarn-site.xml
 (9) 启动 ./sbin/start-yarn.sh
 
* HA HDFS 部署流程（将一个非HA的集群转换为HA的集群）
 (1)在hdfs-site.xml中增加HA的相关配置（**dfs.nameservices**,**dfs.ha.namenodes.[nameservice ID]**,**dfs.namenode.rpc-address.[nameservice ID].[name node ID]**,**dfs.namenode.http-address.[nameservice ID].[name node ID]**,**dfs.namenode.shared.edits.dir**,**dfs.client.failover.proxy.provider.[nameservice ID]**,**dfs.ha.fencing.methods**，**dfs.journalnode.edits.dir**）
 (2)在 core-site.xml中增加相关配置（**fs.defaultFS**）
 (3)将Active的NameNode元数据的目录拷贝到StandBy的NameNode
 (4)执行`hdfs namenode -initializeSharedEdits`初始化JournalNodes
 `(5),(6)为自动Failover的配置`
 (5)在hdfs-site.xml中打开自动failover:
 dfs.ha.automatic-failover.enabled；
 配置zk地址:
 ha.zookeeper.quorum
 (6)初始化ZKFC
 /bin/hdfs zkfc -formatZK
 (7)启动HDFS:start-dfs.sh
 
* HDFS读写的流程 
**读:**Client向NameNode发送读请求，NameNode返回block列表，Client选择离最近的DataNode读取block，如果在读取的过程中失败，就尝试连接别的DataNode进行读取。
**写:**
(1)Client向NameNode发送写请求，Namenode检查目标文件是否已经存在，检查是否可以上传
(2)NameNode返回是否可以上传
(3)Client对文件进行切分，切分为Block，并请求NameNode上传第一个block
(4)namenode返回datanode的服务器
(5)Client请求一台datanode上传数据，构建pipeline，第一个datanode收到请求会继续调用第二个datanode，然后第二个调用第三个datanode，将整个pipeline建立完成，逐级返回客户端
(6)Client以packet为单位传输文件
(7)当一个block传输完成之后，client再次请求namenode上传第二个block的服务器 重复(4)-(6)
 
### Yarn

* 提交任务的流程

(1) Client向RM发送New Application Request请求，RM返回ApplicationID
(2) Client构造并发送ApplicationSubmitContext(包括了调度队列，优先级，用户认证信息，ApplicationID，AM的ContainerLaunchContext(jar包、依赖文件、安全token、启动的shell脚本))
(3) RM在NodeManager上创建container，并启动AM
(4) AM向RM发起注册请求，RM返回集群资源情况
(5) AM向RM请求资源，传递的信息主要包含请求container的列表
(6) RM在收到AM的资源请求后，会根据调度策略，来分配container以满足AM的请求
(7) AM向container所在机器的NM发送ContainerLaunchContext来启动container


---

## Yarn FAQ:

Q1: Yarn各节点的角色及功能？

Q2: Yarn上Application运行流程？

Q3: Yarn如何做资源隔离？

Q4: Vcore vs cores ?

Q5: Yarn如何做资源调度，有哪些调度算法, 如何配置？Yarn队列的作用？

可以分两层，第一小层是YARN的队列，第二小层是队列内的调度。Spark作业提交到不同的队列，通过设置不同队列的minishare、weight等，来实现不同作业调度的优先级，
这一点Spark应用跟其他跑在YARN上的应用并无二致，统一由YARN公平调度。比较好的做法是每个用户单独一个队列，这种配置FAIR调度就是针对用户的了，
可以防止恶意用户提交大量作业导致拖垮所有人的问题。这个配置在hadoop的yarn-site.xml里。

```
<property>
    <name>yarn.resourcemanager.scheduler.class</name>
    <value>org.apache.hadoop.yarn.server.resourcemanager.scheduler.fair.FairScheduler</value>
</property>
```

Q6: Yarn如何做HA?

---

## HDFS FAQ:

Q1: Hdfs各节点的角色及功能？

Q2: Hdfs File的文件结构？

Q3: Hdfs 文件读写的交互流程？

Q4: HDFS 如何做HA?

https://www.ibm.com/developerworks/cn/opensource/os-cn-hadoop-name-node/

Q5: Hdfs文件block的放置策略？

A5: 相同rack 2个，其他rack 1个

Q6: Rack 感知？

A6: 在core-site.xml中配置`net.topology.script.file.name`，指定rack感知脚本.