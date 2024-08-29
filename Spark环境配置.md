## Hadoop 相关

### Hadoop 集群配置
以下是本环境配置的基础：
- ubuntu 20.04
- java 1.8
- hadoop 3.1.3
- scala 2.13.0
- spark 3.3.0
#### Hadoop 基础配置&yarn

在$HADOOP_HOME/etc/hadoop下的如下几个文件中追加以下配置

```xml
<!--core-site.xml-->
<configuration>
    <!-- 指定Name Node的地址 -->
    <property>
            <name>fs.defaultFS</name>
            <value>hdfs://Worker1:8020</value>
    </property>
    <!-- 指定hadoop数据的存储目录 -->
    <property>
            <name>hadoop.tmp.dir</name>
            <value>/usr/local/hadoop/data</value>
    </property>
    <property>
            <name>hadoop.http.staticuser.user</name>
            <value>hadoop</value>
    </property>
</configuration>  

<!--hdfs-site.xml-->
<configuration>
    <!-- nn web端访问地址 -->
    <property>
            <name>dfs.namenode.http-address</name>
            <value>Worker1:9870</value>
    </property>
    <!-- 2nn web端访问地址 -->
    <property>
            <name>dfs.namenode.secondary.http-address</name>
            <value>Worker3:9868</value>
    </property>
</configuration>

<!--yarn-site.xml-->
<!--指定MR走shuffle-->
<property>
        <name>yarn.nodemanager.aux-services</name>
        <value>mapreduce_shuffle</value>
</property>
<!--指定ResourceManager的地址-->
<property>
        <name>yarn.resourcemanager.hostname</name>
        <value>Worker2</value>
</property>
<!--环境变量的继承-->
<property>
    <name>yarn.nodemanager.env-whitelist</name>
    <value>JAVA_HOME,HADOOP_COMMON_HOME,HADOOP_HDFS_HOME,HADOOP_CONF_DIR,CLASSPATH_PREPEND_DISTCACHE,HADOOP_YARN_HOME,HADOOP_MAPRED_HOME</value>
</property>

```

#### Hadoop历史服务进程配置

- 在$HADOOP_HOME/etc/hadoop/mapred-site.xml文件中配置如下

```xml
<property>
        <name>mapreduce.framework.name</name>
        <value>yarn</value>
</property>
<!--历史服务器端地址-->
<property>
        <name>mapreduce.jobhistory.address</name>
        <value>Worker1:10020</value>
</property>
<!--历史服务器web地址-->
<property>
        <name>mapreduce.jobhistory.webapp.address</name>
        <value>Worker1:19888</value>
</property>

```

- 记得分发给集群中其他机器

```sh
# 分发脚本——xsync
#！/bin/bash
#1.判断参数个数
if [ $# -lt 1 ]
then
        echo Not Enough Arguement!
        exit;
fi
#2. 遍历集群所有机器
for host in Worker1 Worker2 Worker3
do
      echo ============  $host   =============
      #3. 遍历所有目录，逐个发送
       for file in $@
       do
          #4.  判断文件是否存在
          if [ -e $file ]
             then
                #5. 获取父目录
                pdir=$(cd -P $(dirname $file); pwd)
                #6. 获取当前文件的名称
                fname=$(basename $file)
                ssh $host "mkdir -p $pdir"
                rsync -av $pdir/$fname $host:$pdir
             else
                 echo  $file does not exists!
        fi
      done
done
```

```sh
# 查看集群java进程脚本——jpsall
#！/bin/bash
for host in Worker1 Worker2 Worker3
do
        echo ================== $host ==================
        ssh $host jps
done
```

- 启动/停止命令如下

```sh
# 在$HADOOP_HOME/bin路径下
maperd --daemon start historyserver	启动
maperd --daemon stop historyserver	停止
```

- 日志聚集功能——集群所有机器的日志聚集起来方便web端查看

```xml
<!--在$HADOOP_HOME/etc/hadoop/yarn-site.xml中配置-->
<!--开启日志聚合功能-->
<property>
        <name>yarn.log-aggregation-enable</name>
        <value>true</value>
</property>
<!--设置日志聚集服务器地址-->
<property>
        <name>yarn.log.server.url</name>
        <value>http://Worker1:19888/jobhistory/logs</value>
</property>
<!--设置日志保留时间为7天-->
<property>
        <name>yarn.log-aggregation.retain-seconds</name>
        <value>604800</value>
</property>
```

### Hadoop集群启动脚本

```sh
# myhadoop.sh——该脚本包含了启动Hadoop集群，Yarn环境，以及启动Hadoop的historyServer

#！/bin/bash
if [ $# -lt 1 ]
then
        echo "No Args Input ..."
        exit;
fi
case $1 in
"start")
                echo "============启动hadoop集群=========="
                ssh Worker1 "/usr/local/hadoop/sbin/start-dfs.sh"
                echo "============启动yarn==============="
                ssh Worker2 "/usr/local/hadoop/sbin/start-yarn.sh"
                echo "============启动hadoop历史服务进程==============="
                ssh Worker1 "/usr/local/hadoop/bin/mapred --daemon start historyserver"
;;
"stop")
                echo "============关闭hadoop历史服务进程==============="
                ssh Worker1 "/usr/local/hadoop/bin/mapred --daemon stop historyserver"
                echo "============关闭yarn==============="
                ssh Worker2 "/usr/local/hadoop/sbin/stop-yarn.sh"
                echo "============关闭hadoop集群=========="
                ssh Worker1 "/usr/local/hadoop/sbin/stop-dfs.sh"
;;
*)
        echo "Input Args Error..."
;;
esac
```

```sh
#hdfs常用指令
hadoop fs -mkdir 名称		创建文件夹
hadoop fs -put 被上传文件 上传到hdfs的位置		上传文件 
<hdfs的相关信息可以进入HDFS的webUI进行查看，具体存储路径在hadoop的core-site.xml进行了配置,具体为：../hadoop/data/dfs/data/current/BP-1026716034-192.168.10.11-1724752970092/current/finalized/subdir0/subdir0>

#hadoop相关WEB UI
HDFS webUI：	NameNode所在节点ip:9870
yarn webUI：	resourceManager所在节点ip:8088
```

### Hadoop 集群测试样例

```sh
#以wordcount为例
hadoop jar share/hadoop/mapreduce/hadoop-mapreduce-examples-3.1.3.jar wordcount /wcinput /wcoutput
#其中/wcinput，/wcoutput都是集群中的路径，前者输入，后者输出
```

### Hadoop 集群启动后没有 datanode 进程

- **问题描述**：第一次启动集群需要格式化NameNode——会产生新的集群ID，因此若是在集群运行过程中需要格式化NN，就会导致NameNode和DataNode的集群id不一样，尝试停止HDFS以及YARN再重启并不能解决问题，**需要先关闭他们再删除集群所有机器上的 data 和 logs 目录，再格式化**

- **解决办法**：
  1. 删除 “../hadoop/data/dfs” 下的两个文件夹 (data、name)
  2. 删除 “../hadoop/data” 下的 /nm-local-dir
  3. 删除 "../hadoop" 下的 /logs
  4. 重新格式化 namenode——在 “/usr/local/hadoop” 下执行 ‘bin/hdfs namenode -format’
- **注意点**
  第一次启动hadoop集群时需要格式化namenode——执行一次format，且在每次格式化之前都需要先停止已经启动的所有namenode和datanode进程，然后在删除data和log数据

## Spark 相关

### Spark 环境配置

- “Hadoop free” binary，需要自己通过配置 `SPARK_DIST_CLASSPATH` 变量，以便可以包含指定版本的Hadoop的相关jar包，比如：spark-2.4.8-bin-without-hadoop-scala-2.12.tgz、spark-2.4.8-bin-without-hadoop.tgz 。

- 由于测试环境虚拟机内存较少，防止进程被杀死，可以尝试在$HADOOP_HOME/etc/hadoop/yarn-site.xml追加配置

  ```xml
  <!--是否启动一个线程检查每个任务正使用的物理内存量，如果任务超出分配值，则直接将其杀掉，默认是 true -->
  <property>
          <name>yarn.nodemanager.pmem-check-enabled</name>
          <value>false</value>
  </property>
  <!--是否启动一个线程检查每个任务正使用的虚拟内存量，如果任务超出分配值，则直接将其杀掉，默认是 true -->
  <property>
          <name>yarn.nodemanager.vmem-check-enabled</name>
          <value>false</value>
  </property>
  ```

  #### 配置Spark历史服务进程

  ```sh
  #在$SPARK_HOME/conf/spark-env.sh中追加
  YARN_CONF_DIR=/usr/local/hadoop/etc/hadoop	#指向yarn配置文件所在位置
  export SPARK_HISTORY_OPTS="
  -Dspark.history.ui.port=18080
  -Dspark.history.fs.logDirectory=hdfs://Worker1:8020/directory
  -Dspark.history.retainedApplications=30"
  ```

  ```xml
  <!--$SPARK_HOME/conf/spark-defaults.conf中追加-->
  spark.eventLog.enabled          true
  #执行文件保存位置，需要该路径提前存在
  spark.eventLog.dir              hdfs://Worker1:8020/directory
  
  spark.yarn.historyServer.address=Worker1:18080
  spark.history.ui.port=18080
  ```

  - **注意点**：需要提前准备好上面执行文件的保存路径(两种方式创建1. 使用hadoop fs -mkdir /directory 指令； 2. 通过Worker2:8088的WEB UI进行创建)

### 启动 Spark 集群

1. 使用群起脚本 myhadoop.sh 启动三台虚拟机构成的模拟集群的hadoop
2. 在 “ /usr/local/spark ” 中启动 ‘sbin/start-all.sh’
3. 在 "/usr/local/spark" 下启动 ‘sbin/start-history-server.sh’

### 测试实例——以SparkPi的计算为例

#### 启动环境

```shell
1. 启动HDFS以及YARN
	myhadoop.sh start
2. 开启Spark的HistoryServer功能
	cd /usr/local/spark
	sbin/start-history-server.sh
```

#### 测试脚本

```sh
# spark-yarn模式——deploy-model可选client或cluster
bin/spark-submit \
--class org.apache.spark.examples.SparkPi \
--master yarn \
--deploy-mode cluster \
./examples/jars/spark-examples_2.13-3.3.0.jar \
10
```

- 两种启动方式的主要区别在于Driver程序的运行节点不同

  |                                                              |
  | ------------------------------------------------------------ |
  | yarn-client(默认)：Driver程序运行在客户端，适用于交互、调试，能立即看到app的输出 |
  | yarn-cluster：Driver程序运行在由ResourceManager启动的APPMaster，适用于生产环境 |

#### 本地测试虚拟机内存不足问题

```xml
尝试在$HADOOP_HOME/etc/hadoop/yarn-site.xml文件中<configuration>标签下追加
<!--是否启动一个线程检查每个任务正使用的物理内存量，如果任务超出分配值，则直接将其杀掉，默认是 true -->
<property>
        <name>yarn.nodemanager.pmem-check-enabled</name>
        <value>false</value>
</property>
<!--是否启动一个线程检查每个任务正使用的虚拟内存量，如果任务超出分配值，则直接将其杀掉，默认是 true -->
<property>
        <name>yarn.nodemanager.vmem-check-enabled</name>
        <value>false</value>
</property>
```



