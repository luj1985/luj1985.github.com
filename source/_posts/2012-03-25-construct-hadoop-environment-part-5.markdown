---
layout: post
title: "Construct Hadoop Environment<br />Part 5 Install Hadoop"
date: 2012-03-25 14:49
comments: true
categories: 
---
Finally got here to start real work :)

# Create VM Template
## Create Linux user to run Hadoop
Here _gentoo_ is the machine name of my VM, and the fully qualified domain name of this VM is _gentoo.hcluster.org_, and the IP address is _192.168.2.103_
```
mkdir /hadoop
useradd -d /hadoop -s /bin/bash -g users -G wheel hadoop
passwd hadoop
chown -R hadoop.users /hadoop
```

## Create SSH public key
```
ssh hadoop@gentoo.hcluster.org
ssh-keygen
cp ~/.ssh/id_rsa.pub ~/.ssh/authorized_keys
```
This VM is used as template, if it can connect to itself, it can connect to any VM which derived from it.
Here is an easy way to copy the SSH public key.
```
ssh-copy-id -i ~/.ssh/id_dsa.pub hadoop@gentoo.hcluster.org
```

Edit _~/.ssh/config_ to prevent host key checking 
```
Host *.hcluster.org
  StrictHostKeyChecking no
  UserKnownHostsFile /dev/null
```
Now it could be able to connect any host which domain is _hcluster.org_ without password checking.

## Install
Download [Apache Hadoop](http://hadoop.apache.org/), currently the latest version is 1.0.1
```
wget http://mirror.bjtu.edu.cn/apache/hadoop/common/stable/hadoop-1.0.1.tar.gz
tar xzf hadoop-1.0.1.tar.gz
ln -s hadoop-1.0.1 hadoop
```

Edit _~/.bash_profile_ to update environment variables
```
export HADOOP_COMMON_HOME=/hadoop/hadoop
export PATH=$HADOOP_COMMON_HOME/bin:$PATH
```

Edit _$HADOOP_COMMON_HOME/conf/hadoop-env.sh_, add _JAVA_HOME_ environment variable setting.
```
export JAVA_HOME=`java-config --jdk-home`
```
# Configure Apache Hadoop environment
Here is my environment 
```
---------------------------------------------------------------
|       Domain Name       |   IP Address  |    MAC Address    |
---------------------------------------------------------------
| jobtracker.hcluster.org | 192.168.2.104 | 00:00:00:00:00:04 |
| namenode1.hcluster.org  | 192.168.2.105 | 00:00:00:00:00:05 |
| namenode2.hcluster.org  | 192.168.2.106 | 00:00:00:00:00:06 |
| datanode1.hcluster.org  | 192.168.2.107 | 00:00:00:00:00:07 |
| datanode2.hcluster.org  | 192.168.2.108 | 00:00:00:00:00:08 |
| datanode3.hcluster.org  | 192.168.2.109 | 00:00:00:00:00:09 |
| datanode4.hcluster.org  | 192.168.2.110 | 00:00:00:00:00:10 |
| datanode5.hcluster.org  | 192.168.2.111 | 00:00:00:00:00:11 |
---------------------------------------------------------------
```

## _$HADOOP_COMMON_HOME/conf/core-site.xml_
Hadoop common properties
```
<?xml version="1.0"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>

<!-- Put site-specific property overrides in this file. -->

<configuration>
    <property>
        <name>fs.default.name</name>
        <value>hdfs://namenode1.hcluster.org</value>
    </property>
</configuration>
```

## _$HADOOP_COMMON_HOME/conf/hdfs-site.xml_
HDFS properties
```
<?xml version="1.0"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>

<!-- Put site-specific property overrides in this file. -->

<configuration>
    <property>
        <name>dfs.replication</name>
        <value>2</value>
    </property>
</configuration>
```

## _$HADOOP_COMMON_HOME/conf/mapred-site.xml_
MapReduce properties
```
<?xml version="1.0"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>

<!-- Put site-specific property overrides in this file. -->

<configuration>
    <property>
        <name>mapred.job.tracker</name>
        <value>jobtracker.hcluster.org:9000</value>
    </property>
</configuration>
```
Must provide port here, seems Hadoop doesn't define the default port for job tracker.

## _$HADOOP_COMMON_HOME/conf/slaves_
```
datanode1.hcluster.org
datanode2.hcluster.org
datanode3.hcluster.org
datanode4.hcluster.org
datanode5.hcluster.org
```

## _$HADOOP_COMMON_HOME/conf/masters_
Secondary name node
```
namenode2.hcluster.org
```

# Clone VM
Remove udev rule file under _/etc/udev/rules.d/_

## 70-persistent-net.rules
When system start it will read udev rules to create device file. 
The device file ``eth0`` was bind to its MAC address. 
That is to say when MAC address changed, the device file name will change too 
(Treat as another network card). This makes _/etc/init.d/net.eth0_ invalid.

These rule files can be generated automatically next reboot. So remove this rule file and reboot can adapt MAC changes.

Shutdown this VM, and create several clones to construct Apache Hadoop Cluster

## Change the hostname
Here the VMs are cloned from one template, all the VM hostnames are the same.

During Hadoop execution, job tracker will `map` the task to every task tracker node, when `map` finished it will do the `reduce`, task trackers trying to copy 
data to `reduce` machine, but here Hadoop seems use `hostname` to connect the machine.  
Host Name is different from Domain Name, in order to make Hadoop work, I have to make them same

Edit _/etc/conf.d/hostname_ for every node, then restart hostname service
```
ssh root@jobtracker.hcluster.org "echo hostname=\\\"jobtracker\\\" > /etc/conf.d/hostname;/etc/init.d/hostname restart"
ssh root@namenode1.hcluster.org "echo hostname=\\\"namenode1\\\" > /etc/conf.d/hostname;/etc/init.d/hostname restart"
ssh root@namenode2.hcluster.org "echo hostname=\\\"namenode2\\\" > /etc/conf.d/hostname;/etc/init.d/hostname restart"
ssh root@datanode1.hcluster.org "echo hostname=\\\"datanode1\\\" > /etc/conf.d/hostname;/etc/init.d/hostname restart"
ssh root@datanode2.hcluster.org "echo hostname=\\\"datanode2\\\" > /etc/conf.d/hostname;/etc/init.d/hostname restart"
ssh root@datanode3.hcluster.org "echo hostname=\\\"datanode3\\\" > /etc/conf.d/hostname;/etc/init.d/hostname restart"
ssh root@datanode4.hcluster.org "echo hostname=\\\"datanode4\\\" > /etc/conf.d/hostname;/etc/init.d/hostname restart"
ssh root@datanode5.hcluster.org "echo hostname=\\\"datanode5\\\" > /etc/conf.d/hostname;/etc/init.d/hostname restart"
```

# Start Apache Hadoop cluster
Log into `namenode1` with `hadoop` account
```
ssh hadoop@namenode1.hcluster.org
```
Format namenode and start cluster
```
hadoop namenode -format
start-all.sh
```
after cluster started, view logs to find whether cluster was started successfully.
Here I found job tracker has errors
```
2012-04-07 13:23:26,317 FATAL org.apache.hadoop.mapred.JobTracker: java.net.BindException: Problem binding to jobtracker.hcluster.org/192.168.2.104:9000 : Cannot assign requested address
    at org.apache.hadoop.ipc.Server.bind(Server.java:227)
    at org.apache.hadoop.ipc.Server$Listener.<init>(Server.java:301)
    at org.apache.hadoop.ipc.Server.<init>(Server.java:1483)
    at org.apache.hadoop.ipc.RPC$Server.<init>(RPC.java:545)
    at org.apache.hadoop.ipc.RPC.getServer(RPC.java:506)
    at org.apache.hadoop.mapred.JobTracker.<init>(JobTracker.java:2306)
    at org.apache.hadoop.mapred.JobTracker.<init>(JobTracker.java:2192)
    at org.apache.hadoop.mapred.JobTracker.<init>(JobTracker.java:2186)
    at org.apache.hadoop.mapred.JobTracker.startTracker(JobTracker.java:300)
    at org.apache.hadoop.mapred.JobTracker.startTracker(JobTracker.java:291)
    at org.apache.hadoop.mapred.JobTracker.main(JobTracker.java:4978)
Caused by: java.net.BindException: Cannot assign requested address
    at sun.nio.ch.Net.bind(Native Method)
    at sun.nio.ch.ServerSocketChannelImpl.bind(ServerSocketChannelImpl.java:137)
    at sun.nio.ch.ServerSocketAdaptor.bind(ServerSocketAdaptor.java:77)
    at org.apache.hadoop.ipc.Server.bind(Server.java:225)
    ... 10 more

```
I don't know why this happened, but I could start job tracker on machine `jobtracker.hcluster.org`
```
ssh hadoop@jobtracker.hcluster.org
hadoop-daemon.sh start jobtracker
```

# Test Cluster
Follow the offical Apache Hadoop examples
```
hadoop fs -put conf input
hadoop jar hadoop-examples-1.0.1.jar grep input output 'dfs[a-z.]+'
```
