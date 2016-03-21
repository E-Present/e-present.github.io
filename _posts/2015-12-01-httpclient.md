---
layout: post
title:  "HttpClient使用"
date:   2015-12-01 13:00
categories: Activiti
permalink: /blogs/HttpClient-1
---
# 下面是 httpClinet4.x的使用说明： 
连接池实现目前有两个类：
1.PoolingHttpClientConnectionManager 
   线程安全，此类会在一段时间内保持仅有一个连接。当访问的route发生变化，原来保持连接将被关闭，重新建立对应当前route的连接。若重复创建该连接会出异常。
2.BasicHttpClientConnectionManager 
    线程安全，此类会在一段时间内保持多个连接来对应每一个route，即会缓存每一个route所对应的多个连接。
  setDefaultMaxPerRoute：设置每一个route所对应的最大连接数；
   setMaxTotal：设置此连接池的总大小，用于容纳所有route所对应的连接；

3.MainClientExec
    The last request executor in the HTTP request execution chain that is responsible for execution of request / response exchanges with the opposite endpoint.
4.AbstractConnPool

# 下面是 httpClinet3.x的使用说明：       
    private final static HttpClient client ;
           private static final HttpClientParams httpClientParams = new HttpClientParams();
           private static final HttpConnectionManagerParams CONNECTION_MANAGER_PARAMS
                                                                                                                            = new HttpConnectionManagerParams();
           static{
                     httpClientParams.setConnectionManagerTimeout(5000);
                     httpClientParams.setParameter(HttpMethodParams. HTTP_CONTENT_CHARSET, "UTF-8");
                     CONNECTION_MANAGER_PARAMS.setConnectionTimeout(5000);
                    //solr中的默认配置
                     CONNECTION_MANAGER_PARAMS.setMaxTotalConnections(128);
                     CONNECTION_MANAGER_PARAMS.setDefaultMaxConnectionsPerHost(32);
                   
                  //SimpleHttpConnectionManager simpleHttpConnectionManager = new SimpleHttpConnectionManager();
                   // simpleHttpConnectionManager.setParams( CONNECTION_MANAGER_PARAMS);
                   
                   MultiThreadedHttpConnectionManager manager = new MultiThreadedHttpConnectionManager();
                   manager.setParams( CONNECTION_MANAGER_PARAMS);
                    client = new HttpClient( httpClientParams, manager);
          }
          
以上都是全局属性，上面的属性说明：
1.HttpClient 一个实例代表一个浏览器，执行的GET或POST方法相当于浏览器中的一个窗口。
2.HttpClient 的每个连接都会对应一个socket连接，故可以通过3中的参数来限制socket的数量。
3.HttpConnectionManagerParams 的参数：（在创建新的连接时会更具这两个参数来控制连接的上限），
    setMaxTotalConnections：设置ConnecatePool池中连接的最大数，其是MaxConnectionsPerHost数的总合。
    setDefaultMaxConnectionsPerHost：设置对应每个远程主机创建连接的最大连接数，即当使用同一个HttpClient对多个不一样的远程主 
                                                           机发送请求时，每个远程主机所对应的连接的最大数，也就是一个远程Host所对应的最大连接    
                                                           数。
     故，以上两个参数的设置是相关联的，同时也host的数目有关。
4.MultiThreadedHttpConnectionManager 支持多线程环境的并发，可以创建多条连接，内置连接池；
   SimpleHttpConnectionManager 只有一个单一的连接可用或者复用，没有连接池；
   DummyConnectionManager 此连接只使用一次，没用连接池
    
注意：
     若操作不慎，造成大量创建socket就会导致本地（客户端）的端口号被用完，出现“ java.net.BindException: Address already in use: connect”异常。以windows为例，默认最大数量的短暂 TCP 端口为 5000 ' ，区间为1024-5000。由于大量创建socket的过程中本地地址（localPort）会由jvm自动分配，由于连接释放比较慢，很快就会把这将近4000个端口沾满，从而出现该异常信息。
