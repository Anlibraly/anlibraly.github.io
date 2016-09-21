---
layout: post
title: zookeeper 日志(zookeeper.out&log4j)配置及清理
description: "zookeeper log out log4j"
tags: [Node, zookeeper]
comments: true
share: true
---

使用zookeeper做服务端的服务发现管理及配置中心，在使用时出现过由于zk的日志大小过大塞满磁盘的情况
似的某个redis服务没法正常运行，因此对zk日志的配置及定时清理做个调研。
<!--more-->

1. zk日志.out及log4j日志路径配置
首先修改bin/zkEnv.sh，配置ZOO_LOG_DIR的环境变量，ZOO_LOG_DIR是zookeeper日志输出目录，ZOO_LOG4J_PROP是log4j日志输出的配置：

<pre><code>
	if [ "x${ZOO_LOG_DIR}" = "x" ]  
	then  
	    ZOO_LOG_DIR="$ZOOBINDIR/../logs"  
	fi  
	  
	if [ "x${ZOO_LOG4J_PROP}" = "x" ]  
	then  
	    ZOO_LOG4J_PROP="INFO,ROLLINGFILE"  //ROLLINGFILE —— 日志轮转，避免单一文件过大
	fi  
</code></pre>

2.zookeeper日志定期清理

[参考链接](http://nileader.blog.51cto.com/1381108/932156)
版本3.4.0以上的zk可以设置定期自动清理。在zoo.cfg中配置：

<pre><code>
	autopurge.purgeInterval: 	24*2  , //这个参数指定了清理频率，单位是小时。默认是0，表示不开启自己清理功能。
	autopurge.snapRetainCount:  2	    //这个参数和上面的参数搭配使用，这个参数指定了需要保留的文件数目。默认是保留3个。
</code></pre>

3.配置zookeeper.out的位置及log4j日志输出

(1)zookeeper.out由nohup的输出，也就是zookeeper的stdout和stdeer输出。

在zkServer.sh中：
<pre><code>
	if [ ! -w "$ZOO_LOG_DIR" ] ; then  
	mkdir -p "$ZOO_LOG_DIR"  
	fi  
  
	_ZOO_DAEMON_OUT="$ZOO_LOG_DIR/zookeeper.out"  //日志输出文件路径
  	
  	//nohup日志输出
    nohup $JAVA "-Dzookeeper.log.dir=${ZOO_LOG_DIR}" "-Dzookeeper.root.logger=${ZOO_LOG4J_PROP}" \  
    -cp "$CLASSPATH" $JVMFLAGS $ZOOMAIN "$ZOOCFG" > "$_ZOO_DAEMON_OUT" 2>&1 < /dev/null &  
</code></pre>

(2)log4j日志输出配置
conf/log4j.properties中：
<pre><code>
	#  
	# Add ROLLINGFILE to rootLogger to get log file output  
	#    Log DEBUG level and above messages to a log file  
	log4j.appender.ROLLINGFILE=org.apache.log4j.RollingFileAppender  //日志轮转，DaliyRollingFileAppender —— 按天轮转
	log4j.appender.ROLLINGFILE.Threshold=${zookeeper.log.threshold}  
	log4j.appender.ROLLINGFILE.File=${zookeeper.log.dir}/${zookeeper.log.file}  
</code></pre>
轮转前提需要将(1)里bin/zkEnv.sh中的轮转配置好

4.zk事务日志查看

zookeeper的事务日志通过zoo.cfg文件中的dataLogDir配置项配置：
<pre><code>
	# the directory where the snapshot is stored.
	# do not use /tmp for storage, /tmp here is just
	# example sakes.
	dataDir=/tmp/zookeeper	
</code></pre>
查看事务日志方法：(需要下载slf4j-api-1.6.1.jar包)
 java -classpath .:slf4j-api-1.6.1.jar:zookeeper-3.4.8.jar org.apache.zookeeper.server.LogFormatter /tmp/zookeeper/version-2/xxx.xxx
