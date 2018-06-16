---
layout: post
title: Hadoop-2.7.6分布式搭建
category: 大数据
tags: [Hadoop, Linux]
---

Hadoop的搭建有三种方式，单机、伪分布式版、完全分布式。这篇文章将介绍如何搭建完全分布式的hadoop集群，以内网服务器和腾讯云服务器进行部署
## 基础环境
### 1、服务器配置
- 三台内网服务器配置：
    
    - 系统:ubuntu 18.04；
    - 内存：2G；
    - 硬盘：100G  
    - ip地址：
        - 10.195.208.59 hadoop-master
        - 10.195.208.86 hadoop-slave2
        - 10.195.208.44 hadoop-slave1
- 三台腾讯云服务器配置：
    - 系统：centos 7.4；
    - 内存：2G；
    - 硬盘：50G
    - ip地址：三台服务器的云主机地址
    
### 2、host配置
> $vim /etc/hostname

- 修改各节点机器的主机名。
- 坑：在hadoop中的主机名千万不要以_命名！！！否则报
ERRORorg.apache.hadoop.hdfs.server.namenode.NameNode:java.lang.IllegalArgumentException: Does not contain a valid host;port authority
- 修改完成后reboot，依次修改slave1、2
> $vim /etc/hosts

```
        #内网服务器配置
        127.0.0.1     localhost
        10.195.208.59 hadoop-master
        10.195.208.86 hadoop-slave2
        10.195.208.44 hadoop-slave1
```

---

```
        #腾讯云服务器配置，以master举例，slave节点自己本身的ip地址为自己的内网IP
        127.0.0.1     localhost
        master机的内网IP  hadoop-master
        slave1机的外网IP  hadoop-slave2
        slave2机的外网IP  hadoop-slave1
```
### 3、免密登陆ssh

首先完成本机无密码访问，下面以master节点举例

```
    $ssh-keyen -t rsa
    #将公钥追加到.ssh下的authorized_keys文件，这个文件用于存放用户的公钥
    cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
    #赋予权限，必须是600，因为不能让所有者之外的用户对authorized_keys文件有写权限
    chmod 600 .ssh/authorized_keys
```

    
此时便可以通过ssh对本机进行无密码访问，依次对slave1、2进行相同的设置
    
然后完成主机之间的无密码访问，只需要把自己机器上生成的公钥远程复制到其他节点，这样每次访问都可以通过私钥和公钥匹配来完成权限认证，下面以master节点无密码访问slave1、2举例

```
    $scp root@hadoop-master:/root/.ssh/id_rsa.pub /root/
    将公钥追加到authorized_keys文件
    $cat id_rsa.pub >> .ssh/authorized_keys
    $rm -rf  id_rsa.pub
```

    
此时便可以在master机器上使用ssh hadoop-slave1进行无密码登陆了
    
最后需要完成slave机器无密登陆master，与上述流程无异
```
    $scp root@hadoop-slave1:/root/.ssh/id_rsa.pub /root/
    $cat id_rsa.pub >> .ssh/authorized_keys
    $rm -rf  id_rsa.pub
```

    
至此，便完成了主从的无密登录

## Hadoop环境的搭建

> 安装统一版本的jdk并配置环境变量
    

```
    yum -y install java-1.8.0-openjdk*
    $vim /etc/profile
    export JAVA_HOME=/usr/lib/jvm/java-1.8.0-openjdk
    export PATH=$JAVA_HOME/bin:$PATH
    export CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
    $source /etc/profile
```

    
>配置hadoop-master上的hadoop环境
    
hadoop版本：2.7.6
    
### 1.  下载hadoop安装包，解压并创建基本目录

```
$wget http://apache.claz.org/hadoop/common/hadoop-2.7.6/hadoop-2.7.6.tar.gz
    $tar -xzvf  hadoop-2.7.6.tar.gz    -C /usr/local 
    $mv  hadoop-2.7.6  hadoop
```
### 2.   配置环境变量

```
    $vim /etc/profile
        export HADOOP_HOME=/usr/local/hadoop
        export HADOOP_COMMON_LIB_NATIVE_DIR=/lib/native
        export HADOOP_CONF_DIR=$HADOOP_HOME/etc/hadoop
        export HADOOP_MAPRED_HOME=$HADOOP_HOME
        export HADOOP_COMMON_HOME=$HADOOP_HOME
        export HADOOP_HDFS_HOME=$HADOOP_HOME
        export CLASSPATH=$($HADOOP_HOME/bin/hadoop classpath):$CLASSPATH
        export PATH=$JAVA_HOME/bin:$HADOOP_HOME/bin:$PATH
    $source /etc/profile
```
### 3.   配置hadoop的配置文件

文件都在：/usr/local/hadoop/etc/hadoop路径下
#### 配置core-site.xml文件
    
```
    $cd /usr/local/hadoop/etc/hadoop
    <configuration>
        <property>
            <name>hadoop.tmp.dir</name>
            <value>file:/usr/local/hadoop/tmp</value>
            <description>hadoop数据存储的临时文件夹。</description>
        </property>
        <property>
            <name>fs.defaultFS</name>
            <value>hdfs://hadoop-master:9000</value>
            <description>指定NameNode的IP地址和端口号</description>
        </property>
    </configuration>
```
坑：如没有配置hadoop.tmp.dir参数，系统默认的临时目录为：/tmp/{$user}。而这个目录在每次重启后都会被删除，必须重新执行format才行，否则会出错。

#### 配置hdfs-site.xml
这个是hdfs的核心配置文件

```
    $cd /usr/local/hadoop/etc/hadoop
    <configuration>
    <property>
        <name>dfs.replication</name>
        <value>3</value>
        <description>指定副本冗余</description>
    </property>
    <property>
        <name>dfs.name.dir</name>
        <value>/usr/local/hadoop/hdfs/name</value>
        <description>namenode节点的文件存储目录</description>
    </property>
    <property>
        <name>dfs.data.dir</name>
        <value>/usr/local/hadoop/hdfs/data</value>
        <description>datanode节点的文件存储目录</description>
    </property>
</configuration>
    
```
#### 配置mapred-site.xml
这个是mapreduce的核心配置文件

```
    $cd /usr/local/hadoop/etc/hadoop
    $cp /usr/local/hadoop/etc/hadoop/mapred-site.xml.template /usr/local/hadoop/etc/hadoop/mapred-site.xml  
    $vim /usr/local/hadoop/etc/hadoop/mapred-site.xml
    <configuration>
      <property>
          <name>mapreduce.framework.name</name>
          <value>yarn</value>
      </property>
       <property>
          <name>mapred.job.tracker</name>
          <value>http://hadoop-master:9001</value>
      </property>
    </configuration>
```
#### 配置yarn-site.xml
这个是hadoop2.x后的资源协调组件,将原本Mapreduce的JobTracker哦那个能拆分，交由ResourceManager进行资源管理、ApplicationMater进行任务调度和监控、NodeManaer执行TaskTracker任务。

```
    <configuration>
    <!-- Site specific YARN configuration properties -->
        <property>
            <name>yarn.nodemanager.aux-services</name>
            <value>mapreduce_shuffle</value>
        </property>
        <property>
            <name>yarn.resourcemanager.hostname</name>
            <value>hadoop-master</value>
        </property>
    </configuration>
```
#### 配置master文件
该文件指定namenode节点所在的服务器机器。删除localhost，添加namenode节点的主机名hadoop-master；

```
    $vim /usr/local/hadoop/etc/hadoop/masters
    hadoop-master
```
#### 配置slaves文件（Master主机特有）
指定哪些服务器节点是datanode节点。

```
    $vim /usr/local/hadoop/etc/hadoop/slaves
    hadoop-slave1
    hadoop-slave2
```
将hadoop目录远程复制给从节点，删除slaves文件

```
    $scp -r /usr/local/hadoop hadoop-slave1:/usr/local/
    $ssh hadoop-slave1
    $rm -rf /usr/local/hadoop/etc/hadoop/slaves
```
分别给从节点配置环境变量

## 集群启动与使用
格式化hdfs文件系统，第一次启动服务前执行的操作，以后不需要执行。作用是把原来HDFS中的数据全部清空，然后再格式化并安装一个全新的hdfs。

```
    $cd /usr/local/hadoop
    $.bin/hadoop namenode -format
    
```
这样以后就可以启动hadoop集群了

```
    $./sbin/start-all.sh
```
通过jps查看集群启动情况

```
    #master
    14563 NameNode
    14788 SecondaryNameNode
    21544 Jps
    14972 ResourceManager
    #slave
    10232 DataNode
    10393 NodeManager
    12655 Jps
```
坑：注意要关闭iptables

![image](http://kkxcx.hutwanghui.club/keng3.png)

至此，hadoop分布式搭建就完成了，可以通过hdfs命令和hdfsadmin命令对hdfs进行操作

![image](http://kkxcx.hutwanghui.club/hdfs-complated.png)

