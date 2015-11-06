#随身云服务器环境

## Ucloud-DMP-大数据集群

 + dmp.door	10.10.169.170 
 	+ username: root
 	+ passsword:	ssy#ucloud2015### hadoop cluster
+	hadoop1.dmp.com	10.10.112.249   + 	hadoop2.dmp.com	10.10.112.82   +	hadoop3.dmp.com	10.10.112.111   +	hadoop4.dmp.com	10.10.107.97  ###hbase cluster+	hbase1.dmp.com	10.10.169.239 +	hbase2.dmp.com	10.10.173.73  +	hbase3.dmp.com	10.10.184.178 ###storm cluster
+	10.10.95.131    storm1.dmp.com zk1.dmp.com nimbus.dmp.com +	10.10.34.92     storm2.dmp.com zk2.dmp.com+	10.10.48.188    storm3.dmp.com zk3.dmp.com###mysql+	10.10.169.170  mysql.dmp.azkaban.master


## Ucloud-peacock

### 跳板机

+ 123.59.83.195
	+ username: root
	+ passsword:	ssy#ucloud2015

+ 跳板机访问其它机器如下：

		在跳板机上 可以   ssh etouch@api1.pc.freed.so   免密码登入

### pc--log-dmp
+	10.10.164.5		log-dmp.pc.freed.so

### peacock admin

+	10.10.131.228		admin1.pc.freed.so

#### peacock bg

+	10.10.168.157		bg1.pc.freed.so
+	10.10.198.136		bg2.pc.freed.so

### peacock api
+	10.10.194.51		api1.pc.freed.so
+	10.10.145.95		api2.pc.freed.so
+	10.10.175.130		api3.pc.freed.so
+	10.10.164.108		api4.pc.freed.so
+	10.10.166.99		api5.pc.freed.so
+	10.10.158.86		api6.pc.freed.so
+	10.10.195.194		api7.pc.freed.so
+	10.10.151.147		api8.pc.freed.so

