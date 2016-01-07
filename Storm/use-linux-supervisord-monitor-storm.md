#使用linux的supervisord监督storm的nimbus、ui和supervisor等守护进程


storm的nimbus和ui都有单点问题。supervisor可能在worker负载过高的时候被kill了。要想恢复服务，必须重新启动这些进程，如果在晚上或者节假日出现这种问题，可能大家都在睡眠和度假的喜悦之中，没有时间和环境去完成这样的工作，这个时候给storm配置一个守护进程是非常必要的。我这里使用了linux的supervisor

Supervisord是运行在python环境下的服务监控程序。所以在安装supervisord之前必须有python环境。

如果系统没有python，centos可以通过yum安装python

```
yum install python
```


## 一.安装supervisor:

  我们的线上环境都采用centos，可以通过yum 安装supervisor

	yum info supervisor
	sudo yum install supervisor
	sudo chkconfig supervisord on

## 二.启动和停止supervisor
   
   	sudo /etc/init.d/supervisord start

   	sudo /etc/init.d/supervisord stop
   	

## 三.编写storm的nimbus、ui和supervisor的启动脚本

（1） nimbus-start.sh的内容如下：

	#!/bin/bash
	
	exec /data/dmp/storm/bin/storm nimbus
	echo "storm-nimbus started !`"


 （2）ui-start.sh的内容如下

	#!/bin/bash
	
	exec /data/dmp/storm/bin/storm ui
	echo "storm-ui started."

 （3）storm-supervisor-start.sh的内容如下

	#!/bin/bash
	
	exec /data/dmp/storm/bin/storm supervisor
	echo "storm-supervisor started ."

 (4) 记得给各个文件赋予用户可执行的权限。比如：
 
    chmod 777 nimbus-start.sh
    chmod 777 ui-start.sh
    chmod 777 storm-supervisor-start.sh

四、配置supervisor

  1. 在Nimbus的机器上配置supervisor对nimbus和storm ui的监控
  打开supervisor的配置文件

	vim /etc/supervisor.conf
在稳健的末尾增加内容如下：

		; storm nimbus
		[program:storm-nimbus]
		command=/data/dmp/storm/bin/nimbus-start.sh ;被监控程序制定的运行脚本
		directory=/data/dmp/storm ;被监控的程序的运行路径
		autorestart=true ;被监控程序异常中断是否自动重启
		
		autostart=true;是否随supervisors进程启动自动启动
		startsecs=5;被监控程序启动时持续时间
		startretries=20;被监控程序启动失败重试的次数
		
		stopsignal=QUIT;被监控程序kill的信号
		
		user=root
		
		; storm ui
		[program:storm-ui]
		command=/data/dmp/storm/bin/ui-start.sh
		directory=/data/dmp/storm
		autorestart=true
		autostart=true
		startsecs=5
		startretries=20
		
		user=root

  2. 在storm-supervisor节点配置supervisors对storm-supervisor的监控

		 ;storm supervisor
		[program:storm-supervisor]
		command=/data/dmp/storm/bin/storm-supervisor-start.sh
		directory=/data/dmp/storm
		autorestart=true
		autostart=true
		startsecs=5
		startretries=20
		user=root

五、重启supervisors

	sudo /etc/init.d/supervisord stop
	sudo /etc/init.d/supervisord start

OK。jps查看一下nimbus、ui、supervisor等进程是否都起来了。搞定