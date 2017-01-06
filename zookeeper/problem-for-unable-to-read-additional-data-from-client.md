## EndOfStreamException：Unable to read additional data from client 


今天再检查Zookeeper的日志中，发现了一些异常，异常栈信息如下：

```
EndOfStreamException: Unable to read additional data from client sessionid 0x156d9134037fd95, likely client has closed socket
        at org.apache.zookeeper.server.NIOServerCnxn.doIO(NIOServerCnxn.java:228)
        at org.apache.zookeeper.server.NIOServerCnxnFactory.run(NIOServerCnxnFactory.java:208)
        at java.lang.Thread.run(Thread.java:745)

```


从异常上看，是Socket的IO操作时，数据尚未读取完，然而socket的流已经被关闭了。

网上搜索了该异常，异常出自如下代码块中：

```
	int rc = sock.read(incomingBuffer);  
	if (rc < 0) {  
	  throw new EndOfStreamException(  
	          "Unable to read additional data from client sessionid 0x"  
	          + Long.toHexString(sessionId)  
	          + ", likely client has closed socket");  
	}  

```

说明，在socket没有读取开头的信息，然而另外一段关闭了socket，那么在什么情况下一段会关闭socket，通常来说，socket超时，还有进程宕机这一类问题比较突出。客户端关闭的情况有可能是在项目发布重启，或者代码异常，这一类问题需要客户端的实现和来配合，发布重启不可避免，但是并不会在短时间内连续重启，然而这一段异常在一个小时内，出现很多次，说明客户端进程挂掉的不是主要矛盾，那么socket超时可能是主要矛盾，尝试性对`zookeeper.session.timeout.ms`做一下调大一些。
