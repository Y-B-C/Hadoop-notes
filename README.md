＃Hadoop笔记

1.	克隆虚拟机		
2.	修改克隆虚拟机的静态IP
3.	修改主机名
4.	关闭防火墙
5.	创建atguigu用户
6.	配置atguigu用户具有root权限
7． 在/opt目录下创建文件夹
	（1）在/opt目录下创建（两个）module(解压后所放的位置)、soft(放  tar.gz包  )文件夹
		sudo /opt/module /opt/soft
	（2）修改两个文件夹的所有者：sudo chown 用户名:用户名 文件名1/ 文件名2/
	（3）解压JDK和Hadoop到/opt/module目录下：
		tar -zxvf 要解压的文件 -C /opt/module/
	（4）配置JDK和Hadoop环境变量
		(使用pwd来查看JDK和Hadoop的路径)
		JAVA_HOME=/opt/module/jdk1.8.0_121		（后面放jdk和hadoop路径）
		HADOOP_HOME=/opt/module/hadoop-2.7.2
		PATH=$PATH:$JAVA_HOME/bin:$HADOOP_HOME/bin:$HADOOP_HOME/sbin
		export JAVA_HOME HADOOP_HOME PATH		(提升为全局变量)
	（5）执行最新的文件（刷新）：source /etc/profile        使用jps查看

==================================================================

免密登录-------ssh.用户名@主机名      |        ssh.主机名(默认当前用户)
1.生成公钥和私钥：ssh-keygen -t rsa       （在 .ssh目录下）
2.将公钥拷贝到要免密登录的目标机器上:	ssh-copy-id hadoop1      |      ssh-copy-id 用户名@主机名 
				ssh-copy-id hadoop2
				ssh-copy-id hadoop3
==================================================================

集群分发脚本xsync（远程同步工具）
1.在/home/用户名 目录下创建bin目录，并在bin目录下xsync创建文件
	mkdir bin------cd bin/------touch xsync------vim xsync
-----------------------------------------------------------------
#!/bin/bash
#校验参数是否合法
if(($#==0))
then
        echo 请输入要分发的文件！
        exit;
fi
#拼接要分发文件的绝对路径
dirpath=$(cd `dirname $1`; pwd -P)
filename=`basename $1`

echo 要分发的文件的路径是：$dirpath/$filename

#循环执行rsyn分发文件到集群的每条机器
for((i=1;i<=3;i++))
do
        echo----------------hadoop$i---------------------
        rsync -rvlt $dirpath/$filename ybc@hadoop$i:$dirpath
done						
==================================================================

修改脚本 xsync 具有执行权限
	chmod u+x xsync

基本语法:	  rsync 要拷贝的文件路径/名称

==================================================================

配置集群
进入：		cd /opt/module/hadoop-2.7.2/etc/hadoop
配置文件：	vim core-site.xml
-------------------------------------------------------------------
解说：在第一台机器启动NameNode
<!-- 指定HDFS中NameNode的地址 -->
<property>
                <name>fs.defaultFS</name>
      <value>hdfs://hadoop1:9000</value>
</property>

<!-- 指定Hadoop运行时产生文件的存储目录 -->
<property>
                <name>hadoop.tmp.dir</name>
                <value>/opt/module/hadoop-2.7.2/data/tmp</value>
</property>
==================================================================
配置文件：vim hdfs-site.xml
解说：SecondaryNameNode在第三台机器运行
-----------------------------------------------------------------------
<!-- 指定Hadoop辅助名称节点主机配置 -->
<property>
      <name>dfs.namenode.secondary.http-address</name>
      <value>hadoop3:50090</value>
</property>
==================================================================
配置文件：vim yarn-site.xml
解说：ResourceManager在第二台机器运行
-----------------------------------------------------------------------
<property>
                <name>yarn.nodemanager.aux-services</name>
                <value>mapreduce_shuffle</value>
</property>

<!-- 指定YARN的ResourceManager的地址 -->
<property>
                <name>yarn.resourcemanager.hostname</name>
                <value>hadoop2</value>
</property>
<!-- 日志聚集功能使能 -->
<property>
<name>yarn.log-aggregation-enable</name>
<value>true</value>
</property>

<!-- 日志保留时间设置7天 -->
<property>
<name>yarn.log-aggregation.retain-seconds</name>
<value>604800</value>
</property>
==================================================================
去掉mapred-site.xml后面的template
mv mapred-site.xml.template mapred-site.xml
配置文件：vim mapred-site.xml
------------------------------------------------------------------------
<property>
                <name>mapreduce.framework.name</name>
                <value>yarn</value>
</property>
<property>
<name>mapreduce.jobhistory.address</name>
<value>hadoop1:10020</value>
</property>
<property>
    <name>mapreduce.jobhistory.webapp.address</name>
    <value>hadoop1:19888</value>
</property>
<!--第三方框架使用yarn计算的日志聚集功能 -->
<property>         <name>yarn.log.server.url</name>         <value>http://hadoop1:19888/jobhistory/logs</value> </property>

==================================================================
返回到module目录向个个机器分发hadoop-2.7.2/
xsync hadoop-2.7.2/

配置文件：vim .bashrc (返回到~目录再执行)
加上：source /etc/profile
分发到个个机器
xsync .bashrc
==================================================================
xcall----->表示批量执行命令          执行命令----->xcall 加要执行的命令
在bin目录上编辑
vim xcall
-------------------------------------------------------------------------------------
#!/bin/bash
#在集群的所有机器上批量执行同一条命令
if(($#==0))
then
        echo 请输入您要操作的命令！
        exit
fi

echo 要执行的命令是$*

#循环执行此命令
for((i=1;i<=3;i++))
do
        echo-----------------------hadoop$i--------------------
        ssh hadoop$i $*
done
----------------------------------------------------------------------------------------
修改执行权限
chmod u+x xcall
-----------------------------------------------------------------------------------------
配置群起群停的脚本（那台机器操作就在那台机器配置）
cd /opt/module/hadoop-2.7.2/etc/hadoop
vim slaves
删掉其它的，写入要群启群停的机器主机名
类似：
hadoop1
hadoop2
hadoop3
注意：除了写主机名其它的空格之类的符号都不能有
------------------------------------------------------------------------------------------
如果集群是第一次启动，需要在NN所配置的节点上进行格式化NameNode
（当前配置在第一台机器，使用只能在第一台机器执行此命令）
hadoop namenode -format
====================================备注===========================================
分别启动/停止HDFS组件------>start(启动) stop(暂停)
hadoop-daemon.sh  start namenode
hadoop-daemon.sh  start datanode
hadoop-daemon.sh  start secondarynamenode

启动/停止YARN
yarn-daemon.sh start resourcemanager
yarn-daemon.sh start nodemanager
---------------整体启动---------------------
整体启动/停止HDFS---配置ssh是前提
start-dfs.sh   /  stop-dfs.sh

整体启动/停止YARN---配置ssh是前提
start-yarn.sh  /  stop-yarn.sh

全部启动/停止
start-all.sh--------->包括YARN和HDFS
=============================================================

停止防火墙的命令：sudo service iptables stop/status(查看防火墙状态)
设置开机不启动防火墙：sudo chkconfig iptables off
创建目录：hadoop fs -mkdir /要创建的目录名
上传文件：hadoop fs -put 文件名 /目录/

scp -r 源文件的用户@主机名：源文件目录  目标文件的用户名@主机名：目标文件路径



