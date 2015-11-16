# sqoop-1.4.6 安装

环境说明，测试的hadoop环境为Hadoop-2.6的Apache版本。

1. 下载sqoop安装包。安装包需要根据hadoop版本来确定安装包，我所使用的环境是hadoop－2.6。因此选择sqoop-1.4.6.bin__hadoop-2.0.4-alpha.tar.gz；不选择sqoop-1.9的原因是sqoop－1.9目前还是开发版本，并不适合。
	
	```
	wget http://apache.fayea.com/sqoop/1.4.6/sqoop-1.4.6.bin__hadoop-2.0.4-alpha.tar.gz
	```
	
2. 解压并且重命名为sqoop-1.4.6

 	```
 	tar -zxvf sqoop-1.4.6.bin__hadoop-2.0.4-alpha.tar.gz
 	mv sqoop-1.4.6.bin__hadoop-2.0.4 sqoop-1.4.6
 	```	
 
3. 配置环境变量。

	```
	vi /etc/profile
	exportSQOOP_HOME=/home/peacock/sqoop-1.4.6
	export PATH=$PATH:$SQOOP_HOME/bin
	``` 	
	保存后，使用source命令，重新加载一下profile中的配置，使得配置生效。
	```
	source /etc/profile
	```
	
4. 修改配置文件（修改$SQOOP_HOME/bin/configure-sqoop）

	注释掉没有用到的服务的检查。通常来说需要注释掉HCatalog、Accumulo检查(除非你准备使用HCatalog，Accumulo等HADOOP上的组件)
	
	```
		##Moved to be a runtime check in sqoop.
		#if[ ! -d "${HCAT_HOME}" ]; then
		#  echo "Warning: $HCAT_HOME does notexist! HCatalog jobs will fail."
		#  echo 'Please set $HCAT_HOME to the root ofyour HCatalog installation.'
		#fi
			
		#if[ ! -d "${ACCUMULO_HOME}" ]; then
		#  echo "Warning: $ACCUMULO_HOME does notexist! Accumulo imports will fail."
		#  echo 'Please set $ACCUMULO_HOME to the rootof your Accumulo installation.'
		#fi
			
		#Add HCatalog to dependency list
		#if[ -e "${HCAT_HOME}/bin/hcat" ]; then
		# TMP_SQOOP_CLASSPATH=${SQOOP_CLASSPATH}:`${HCAT_HOME}/bin/hcat-classpath`
		#  if [ -z "${HIVE_CONF_DIR}" ]; then
		#   TMP_SQOOP_CLASSPATH=${TMP_SQOOP_CLASSPATH}:${HIVE_CONF_DIR}
		#  fi
		#  SQOOP_CLASSPATH=${TMP_SQOOP_CLASSPATH}
		#fi
			
		#Add Accumulo to dependency list
		#if[ -e "$ACCUMULO_HOME/bin/accumulo" ]; then
		#  for jn in `$ACCUMULO_HOME/bin/accumuloclasspath | grep file:.*accumulo.*jar |cut -d':' -f2`; do
		#    SQOOP_CLASSPATH=$SQOOP_CLASSPATH:$jn
		#  done
		#  for jn in `$ACCUMULO_HOME/bin/accumuloclasspath | grep file:.*zookeeper.*jar |cut -d':' -f2`; do
		#    SQOOP_CLASSPATH=$SQOOP_CLASSPATH:$jn
		#  done
		#fi
		
		# export HCAT_HOME=
		# export ACCUMULO_HOME=
	```
	这里注释的可能不完全，请仔细检查。如果没有注释干净，可能在数据库连接的时候，出现无法获取系统语言变量值的异常。
  	 		

5. 将Mysql的jdbc连接驱动包添加到SQOOP_HOME/lib下，mysql-connector-java-5.1.30.jar一定要选择经过生产环境的测试的jar，否则可能出现异常。如果出现连接的异常，极有可能是mysql-connector-java的版本有问题。
6. 启动

```
   sqoop help
```
启动如果出现Error: Could not find or load main class org.apache.sqoop.Sqoop，这是因为找不到sqoop-1.4.6.jar文件导致的。

解决方案：

打开sqoop脚本。找到```exec ${HADOOP_COMMON_HOME}/bin/hadoop org.apache.sqoop.Sqoop "$@"```，显示的指定sqoop-1.4.6.jar的位置。

修改$SQOOP_HOME/bin/sqoop脚本：

```
* 修改前：
exec ${HADOOP_COMMON_HOME}/bin/hadoop org.apache.sqoop.Sqoop "$@"
* 修改后：
exec ${HADOOP_COMMON_HOME}/bin/hadoop jar $SQOOP_HOME/sqoop-1.4.6.jar org.apache.sqoop.Sqoop "$@"
```



   