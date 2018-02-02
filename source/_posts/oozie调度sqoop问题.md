---
layout: draft
title: 采用hue使用oozie调度sqoop问题
date: 2018-01-27 14:16:38
tags:
---

##环境

软件|版本|备注
---|---|---|
HDP|2.6.3|2rm+3nm(=48g,24c)+1client|
Hue|4.1.0|client节点|
centos|7.0||

其他：
1. slave2+master1在一个母机
2. slave1,slave3,master2在一个母机
3. 母机之间是万兆网卡


##部署过程

1. 运维帮忙安装必要的依赖包(curl,jdk8+,ntp,openssl 1.01,python 2.7.x ,rpm,scp,tar,unzip,wget,yum)
2. 通过ambari先部署的hdp基础的hadoop环境，然后部署的oozie，保证配置，软件环境等一致。


##调试过程
在Hue的editor下使用sqoop执行 

```bash
import   --connect jdbc:mysql://xx:3306/xx --username xx --password xx --table weight --hive-table tmp.test --hive-import  --hive-overwrite -m 1
```


###问题1：只安装了jre没有安装jdk
主要报错内容 

```bash 
13/04/19 10:35:24 ERROR tool.ImportTool: Encountered IOException running import job: java.io.IOException: Could not start Java compiler.
```

解决思路
1. 首先安装java-1.7.0-openjdk-devel `yum install -y java-1.7.0-openjdk-devel`
2. 整个平台的jdk路径由ambari-server分发,配置ambari.properties中java.home=/usr/lib/jvm/java
3. 重启整个平台，ambari分发配置



###问题2：hiveconf class not find

主要报错内容

```bash
    Could not load org.apache.hadoop.hive.conf.HiveConf. Make sure HIVE_CONF_DIR is set correctly.
     
    Encountered IOException running import job: java.io.IOException: java.lang.ClassNotFoundException: org.apache.hadoop.hive.conf.HiveConf
    
```

解决思路
1. oozie在安装的时候回自动在hdfs中上传对应的jar包，更加action的类型取classpath，sqoop该action没有将hive的jar包含进来
2. 可以通过设置oozie.action.sharelib.for.{action}参数来指定某个action的全局classpath，所以设置oozie.action.sharelib.for.sqoop=sqoop,hive
3. 重启oozie服务


###问题3：system kill 没有hive-site.xml
主要报错内容:

```bash
Main class [org.apache.oozie.action.hadoop.SqoopMain], exit code [1]
```
只有这么一句话没有其他任何对应的提示，通过google发现很多报错都是这个信息，但是对应不同的具体错误

解决方案
1. 尝试直接用oozie调度hive，可以成功的写入hive
2. 发现每次oozie调度sqoop的时候都能够写入hdfs，说明最后一步写入hive出错
3. 结合网上的资料发现需要制定hive-site.xml,tez-site.xml
4. 考虑到都是新人就直接全局把hive-site.xml,tez-site.xml放到sqoop这个sharelib中，并update
5. 重启oozie服务

###问题4：hue默认不开启rerun参数

主要报错内容

```bash
Main class [org.apache.oozie.action.hadoop.SqoopMain], exit code [1]
```
和上面的完全一致，但是偶尔成功，通过观察发现sqoop这个job如果是在slave2节点上启动的时候都成功，但是如果是其他节点启动sqoop的container就会失败，于是认为是节点之间的环境不一致，但是ambari统一部署的不会存在环境问题，怀疑是网络不稳定，同时多次试验发现slave2也会出现失败，其他节点也会有极小的概率成功

解决过程：
1. oozie的报错真的很不友好，基本上靠猜。
2. 想到会不会是hue和hdp水土不服，毕竟之前有hive的教训(hdp中要加入hive.server2.parallel.ops.in.session=true，否则hue中无法正常使用hive)
3. 尝试使用ambari的workflow view来编写调度，一样的命令发现可以执行，发现2者的workflow不完全一直，action的版本上面有一些差异
4. 对应在hue中修改一致workflow，但是发现还是不行，继续查找到job.properties中的重要信息oozie.wf.rerun.failnodes=true
5. 参考[oozie官网](https://oozie.apache.org/docs/3.1.3-incubating/DG_WorkflowReRun.html)说明也是很少
6. hue中workflow加上该参数，发现可以执行
7. 考虑到sqoop前期用的很多，hue中的sqoop editor没有可以设置参数的地方，于是全局设置oozie.wf.rerun.failnodes=true
8. 重启服务，一切正常

##后话
拍错是一个细心的活，尤其是对于这种报错信息极少的，需要靠经验来猜。