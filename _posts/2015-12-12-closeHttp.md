---
layout: post
title:  "HTTP连接 关闭"
date:   2015-03-02 13:00
categories: java
permalink: /blogs/close-http
---
http协议一是建立在TCP协议之上的。所以一个http连接的也要经过三次握手才能建立。关闭连接时需要四次握手。如下图：
![](http://e-present.github.io/images/httpclose.png)

一般服务器连接的配置：

	1. Timeout 30  
	2. KeepAlive On    #表示服务器端不会主动关闭链接  
	3. MaxKeepAliveRequests 100  
	4. KeepAliveTimeout 180   

这个配置就会导致该连接在闲置180秒后才会关闭。

以HttpClient 为例建立http：
客户端主动发起关闭连接的三种方式：
方法一： 
把事例代码中的第一行实例化代码改为如下即可，在method.releaseConnection();之后connection manager会关闭connection 。 
Java代码  

	1. HttpClient client = new HttpClient(new HttpClientParams(),new SimpleHttpConnectionManager(true) );  


方法二： 
实例化代码使用：HttpClient client = new HttpClient(); 
在method.releaseConnection();之后加上 
Java代码  

	1. ((SimpleHttpConnectionManager)client.getHttpConnectionManager()).shutdown();  


shutdown源代码很简单，看了一目了然 
Java代码  

	1. public void shutdown() {  
	2.     httpConnection.close();  
	3. }  


方法三： 
实例化代码使用：HttpClient client = new HttpClient(); 
在method.releaseConnection();之后加上 
client.getHttpConnectionManager().closeIdleConnections(0);此方法源码代码如下： 
Java代码  

	1. public void closeIdleConnections(long idleTimeout) {  
	2.     long maxIdleTime = System.currentTimeMillis() - idleTimeout;  
	3.     if (idleStartTime <= maxIdleTime) {  
	4.         httpConnection.close();  
	5.     }  
	6. }  


服务器端主动关闭连接：
只需要在HttpMethod method = new GetMethod("http://www.apache.org");加上一行HTTP头的设置即可 
Java代码  

	1. method.setRequestHeader("Connection", "close");  

看一下HTTP协议中关于这个属性的定义： 
HTTP/1.1 defines the "close" connection option for the sender to signal that the connection will be closed after completion of the response. For example：
       Connection: close 

总结：
现在再说一下客户端关闭链接和服务器端关闭链接的区别。如果采用客户端关闭链接的方法，在客户端的机器上使用netstat –an命令会看到很多TIME_WAIT的TCP链接。如果服务器端主动关闭链接这种情况就出现在服务器端。 
The TIME_WAIT state is a protection mechanism in TCP. The side that closes a socket connection orderly will keep the connection in state TIME_WAIT for some time, typically between 1 and 4 minutes. 
TIME_WAIT的状态会出现在主动关闭链接的这一端。TCP协议中TIME_WAIT状态主要是为了保证数据的完整传输。


另外强调一下使用上面这些方法关闭链接是在我们的应用中明确知道不需要重用链接时可以主动关闭链接来释放资源。如果你的应用是需要重用链接的话就没必要这么做，使用原有的链接还可以提供性能。

    TCP连接关闭：       假设Client端发起中断连接请求，也就是发送FIN报文。Server端接到FIN报文后，意思是说"我Client端没有数据要发给你了"，但是如果你还有数据
