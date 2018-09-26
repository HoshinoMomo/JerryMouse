tomcat架构
很多开源应用服务器都是集成tomcat作为web container的，而且对于tomcat的servlet container这部分代码很少改动。这样，这些应用服务器的性能基本上就取决于Tomcat处理HTTP请求的connector模块的性能。本文首先从应用层次分析了tomcat所有的connector种类及用法，接着从架构上分析了connector模块在整个tomcat中所处的位置，最后对connector做了详细的源代码分析。并且我们以Http11NioProtocol为例详细说明了tomcat是如何通过实现ProtocolHandler接口而构建connector的。

 一、Connector介绍

1. Connector

在Tomcat架构中，Connector主要负责处理与客户端的通信。Connector的实例用于监听端口，接受来自客户端的请求并将请求转交给Engine处理。同时将来自Engine的答复返回给客户端。

2. Connector的种类

Tomcat源码中与connector相关的类位于org.apache.coyote包中，Connector分为以下几类：

Http Connector, 基于HTTP协议，负责建立HTTP连接。它又分为BIO Http Connector与NIO Http Connector两种，后者提供非阻塞IO与长连接Comet支持。默认情况下，Tomcat使用的就是这个Connector。

AJP Connector, 基于AJP协议，AJP是专门设计用来为tomcat与http服务器之间通信专门定制的协议，能提供较高的通信速度和效率。如与Apache服务器集成时，采用这个协议。

APR HTTP Connector, 用C实现，通过JNI调用的。主要提升对静态资源(如HTML、图片、CSS、JS等)的访问性能。现在这个库已独立出来可用在任何项目中。Tomcat在配置APR之后性能非常强劲。

具体地，Tomcat7中实现了以下几种Connector：

org.apache.coyote.http11.Http11Protocol : 支持HTTP/1.1 协议的连接器。

org.apache.coyote.http11.Http11NioProtocol : 支持HTTP/1.1 协议+New IO的连接器。

org.apache.coyote.http11.Http11AprProtocol : 使用APR（Apache portable runtime)技术的连接器,利用Native代码与本地服务器（如linux）来提高性能。

（以上三种Connector实现都是直接处理来自客户端Http请求，加上NIO或者APR）

org.apache.coyote.ajp.AjpProtocol：使用AJP协议的连接器，实现与web server（如Apache httpd）之间的通信

org.apache.coyote.ajp.AjpNioProtocol：SJP协议+ New IO

org.apache.coyote.ajp.AjpAprProtocol：AJP + APR

（这三种实现方法则是与web server打交道，同样加上NIO和APR）

当然，我们可以通过实现ProtocolHandler接口来定义自己的Connector。详细的实现过程请看文章后文。

3. Connector的配置

对Connector的配置位于conf/server.xml文件中，内嵌在Service元素中，可以有多个Connector元素。

(1) BIO HTTP/1.1 Connector配置

一个典型的配置如下：　　

<Connector  port=”8080”  protocol=”HTTP/1.1”  maxThreads=”150”  conn ectionTimeout=”20000”   redirectPort=”8443” />
其它一些重要属性如下：
acceptCount : 接受连接request的最大连接数目，默认值是10
address : 绑定IP地址，如果不绑定，默认将绑定任何IP地址
allowTrace : 如果是true,将允许TRACE HTTP方法
compressibleMimeTypes : 各个mimeType, 以逗号分隔，如text/html,text/xml
compression : 如果带宽有限的话，可以用GZIP压缩
connectionTimeout : 超时时间，默认为60000ms (60s)
maxKeepAliveRequest : 默认值是100
maxThreads : 处理请求的Connector的线程数目，默认值为200
如果是SSL配置，如下：

<Connector port="8181" protocol="HTTP/1.1" SSLEnabled="true" 
    maxThreads="150" scheme="https" secure="true" 
    clientAuth="false" sslProtocol = "TLS" 
    address="0.0.0.0" 
    keystoreFile="E:/java/jonas-full-5.1.0-RC3/conf/keystore.jks" 
    keystorePass="changeit" />
其中，keystoreFile为证书位置，keystorePass为证书密码

(2) NIO HTTP/1.1 Connector配置

<Connector port=”8080” protocol=”org.apache.coyote.http11.Http11NioProtocol” maxThreads=”150” connectionTimeout=”20000” redirectPort=”8443”/>
(3) Native APR Connector配置

ARP是用C/C++写的，对静态资源(HTML，图片等)进行了优化。所以要下载本地库tcnative-1.dll与openssl.exe，将其放在%tomcat%\bin目录下。
下载地址是：http://tomcat.heanet.ie/native/1.1.10/binaries/win32/

在server.xml中要配置一个Listener,如下图。这个配置tomcat是默认配好的。
<!--APR library loader. Documentation at /docs/apr.html --> 
<Listener className="org.apache.catalina.core.AprLifecycleListener" SSLEngine="on" />
配置使用APR connector
<Connector port=”8080” protocol=”org.apache.coyote.http11.Http11AprProtocol” 
maxThreads=”150” connectionTimeout=”20000” redirectPort=”8443”
 
如果配置成功，启动tomcat,会看到如下信息：
org.apache.coyote.http11.Http11AprProtocol init
(4)  AJP Connector配置：
<Connector port="8009" protocol="AJP/1.3" redirectPort="8443" />
二、Connector在Tomcat中所处的位置

1. Tomcat架构



>>>Server(服务器)是Tomcat构成的顶级构成元素，所有一切均包含在Serv    erz中，Server的实现类StandardServer可以包含一个到多个Services;>>>>Service的实现类为StandardService调用了容器(Container)     接口，其实是调用了Servlet Engine(引擎)，而且StandardService类中也   指明了该Service归属的Server;

>>>Container，引擎(Engine)、主机(Host)  上下文(Context)和Wraper均继承自Container接口，所以它们都是容器       但是，它们是有父子关系的，在主机(Host)、上下文(Context)和引擎(En      gine)这三类容器中，引擎是顶级容器，直接包含是主机容器，而主机容       器又包含上下文容器，所以引擎、主机和上下文从大小上来说又构成父子     关系,虽然它们都继承自Container接口。
>>>连接器(Connector)将Service和Container连接起来，首先它需要注册到一个Service，它的作用就是把来自客户端的请求转发到Container(容器)，这就是它为什么称作连接器的原因。
故我们从功能的角度将Tomcat源代码分成5个子模块，它们分别是：

Jsper子模块：这个子模块负责jsp页面的解析、jsp属性的验证，同时也负责将jsp页面动态转换为java代码并编译成class文件。在Tomcat源代码中，凡是属于org.apache.jasper包及其子包中的源代码都属于这个子模块;

Servlet和Jsp规范的实现模块：这个子模块的源代码属于javax.servlet包及其子包，如我们非常熟悉的javax.servlet.Servlet接口、javax.servet.http.HttpServlet类及javax.servlet.jsp.HttpJspPage就位于这个子模块中;

Catalina子模块：这个子模块包含了所有以org.apache.catalina开头的java源代码。该子模块的任务是规范了Tomcat的总体架构，定义了Server、Service、Host、Connector、Context、Session及Cluster等关键组件及这些组件的实现，这个子模块大量运用了Composite设计模式。同时也规范了Catalina的启动及停止等事件的执行流程。从代码阅读的角度看，这个子模块应该是我们阅读和学习的重点。

Connector子模块：如果说上面三个子模块实现了Tomcat应用服务器的话，那么这个子模块就是Web服务器的实现。所谓连接器(Connector)就是一个连接客户和应用服务器的桥梁，它接收用户的请求，并把用户请求包装成标准的Http请求(包含协议名称，请求头Head，请求方法是Get还是Post等等)。同时，这个子模块还按照标准的Http协议，负责给客户端发送响应页面，比如在请求页面未发现时，connector就会给客户端浏览器发送标准的Http 404错误响应页面。

Resource子模块：这个子模块包含一些资源文件，如Server.xml及Web.xml配置文件。严格说来，这个子模块不包含java源代码，但是它还是Tomcat编译运行所必需的。

2.Tomcat运行流程

假设来自客户的请求为：http://localhost:8080/test/index.jsp.请求被发送到本机端口8080，被在那里侦听的Coyote HTTP/1.1 Connector获得,然后
 

Connector把该请求交给它所在的Service的Engine来处理，并等待Engine的回应
Engine获得请求localhost:8080/test/index.jsp，匹配它所有虚拟主机Host
Engine匹配到名为localhost的Host(即使匹配不到也把请求交给该Host处理，因为该Host被定义为该Engine的默认主机)
localhost Host获得请求/test/index.jsp，匹配它所拥有的所有Context
Host匹配到路径为/test的Context(如果匹配不到就把该请求交给路径名为""的Context去处理)
path="/test"的Context获得请求/index.jsp，在它的mapping table中寻找对应的servlet
Context匹配到URL PATTERN为*.jsp的servlet，对应于JspServlet类，构造HttpServletRequest对象和HttpServletResponse对象，作为参数调用JspServlet的doGet或doPost方法
Context把执行完了之后的HttpServletResponse对象返回给Host
Host把HttpServletResponse对象返回给Engine
Engine把HttpServletResponse对象返回给Connector
Connector把HttpServletResponse对象返回给客户browser
三、Connector源码分析

1 Tomcat的启动分析与集成设想

我们知道，启动tomcat有两种方式：双击bin/startup.bat、运行bin/catalina.bat run。它们对应于Bootstrap与Catalina两个类，我们现在只关心Catalina这个类，这个类使用Apache Digester解析conf/server.xml文件生成tomcat组件，然后再调用Embedded类的start方法启动tomcat。

所以，集成Tomcat的方式就有以下两种了：

①沿用tomcat自身的server.xml

②自己定义一个xml格式来配置tocmat的各参数，自己再写解析这段xml，然后使用tomcat提供的API根据这些xml来生成Tomcat组件，最后调用Embedded类的start方法启动tomcat

个人觉得第一种方式要优越，给开发者比较好的用户体验，如果使用这种，直接模仿Catalina类的方法即可实现集成。

目前，JOnAS就使用了这种集成方式，JBoss、GlassFish使用的第二种自定义XML的方式。

2. Connector类图与顺序图

　　

　

　　

从上面二图中我们可以得到如下信息：

Tomcat中有四种容器(Context、Engine、Host、Wrapper)，前三者常见，第四个不常见但它也是实现了Container接口的容器

如果要自定义一个Connector的话，只需要实现org.apache.coyote.ProtocolHander接口,该接口定义如下：

 

[java] view plain copy

<span style="font-size:14px;">/* 
 *  Licensed to the Apache Software Foundation (ASF) under one or more 
 *  contributor license agreements.  See the NOTICE file distributed with 
 *  this work for additional information regarding copyright ownership. 
 *  The ASF licenses this file to You under the Apache License, Version 2.0 
 *  (the "License"); you may not use this file except in compliance with 
 *  the License.  You may obtain a copy of the License at 
 * 
 *      http://www.apache.org/licenses/LICENSE-2.0 
 * 
 *  Unless required by applicable law or agreed to in writing, software 
 *  distributed under the License is distributed on an "AS IS" BASIS, 
 *  WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. 
 *  See the License for the specific language governing permissions and 
 *  limitations under the License. 
 */  
  
package org.apache.coyote;  
  
import java.util.concurrent.Executor;  
  
  
/** 
 * Abstract the protocol implementation, including threading, etc. 
 * Processor is single threaded and specific to stream-based protocols, 
 * will not fit Jk protocols like JNI. 
 * 
 * This is the main interface to be implemented by a coyote connector. 
 * Adapter is the main interface to be implemented by a coyote servlet 
 * container. 
 * 
 * @author Remy Maucherat 
 * @author Costin Manolache 
 * @see Adapter 
 */  
public interface ProtocolHandler {   //Tomcat 中的Connector实现了这个接口  
  
    /** 
     * The adapter, used to call the connector. 
     */  
    public void setAdapter(Adapter adapter);  
    public Adapter getAdapter();  
  
  
    /** 
     * The executor, provide access to the underlying thread pool. 
     */  
    public Executor getExecutor();  
  
  
    /** 
     * Initialise the protocol. 
     */  
    public void init() throws Exception;  
  
  
    /** 
     * Start the protocol. 
     */  
    public void start() throws Exception;  
  
  
    /** 
     * Pause the protocol (optional). 
     */  
    public void pause() throws Exception;  
  
  
    /** 
     * Resume the protocol (optional). 
     */  
    public void resume() throws Exception;  
  
  
    /** 
     * Stop the protocol. 
     */  
    public void stop() throws Exception;  
  
  
    /** 
     * Destroy the protocol (optional). 
     */  
    public void destroy() throws Exception;  
  
  
    /** 
     * Requires APR/native library 
     */  
    public boolean isAprRequired();  
}  
</span>  

自定义Connector时需实现的ProtoclHandler接口
 

Tomcat以HTTP(包括BIO与NIO)、AJP、APR、内存四种协议实现了该接口(它们分别是：AjpAprProtocol、AjpProtocol、Http11AprProtocol、Http11NioProtocol、Http11Protocal、JkCoyoteHandler、MemoryProtocolHandler)，要使用哪种Connector就在conf/server.xml中配置，在Connector的构造函数中会通过反射实例化所配置的实现类：

<Connector port="8181" 
   protocol="org.apache.coyote.http11.Http11AprProtocol " />
 
3.Connector的工作流程
下面我们以org.apache.coyote.http11.Http11AprProtocol为例说明Connector的工作流程。
①它将工作委托给AprEndpoint类。
[java] view plain copy

<span style="font-size:14px;">    public Http11AprProtocol() {  
        endpoint = new AprEndpoint();                    // 主要工作由AprEndpoint来完成  
        cHandler = new Http11ConnectionHandler(this);    // inner class  
        ((AprEndpoint) endpoint).setHandler(cHandler);</span>  
②在AprEndpoint.Acceptor类中的run()方法会接收一个客户端新的连接请求.

 

[java] view plain copy

/** 
 * The background thread that listens for incoming TCP/IP connections and 
 * hands them off to an appropriate processor. 
 */  
protected class Acceptor extends AbstractEndpoint.Acceptor {  
  
    private final Log log = LogFactory.getLog(AprEndpoint.Acceptor.class);  
  
    @Override  
    public void run() {  
  
        int errorDelay = 0;  
  
        // Loop until we receive a shutdown command  
        while (running) {  


 

③在AprEndpoint类中，有一个内部接口Handler，该接口定义如下：

 

[java] view plain copy

<span style="font-size:14px;">/** 
     * Bare bones interface used for socket processing. Per thread data is to be 
     * stored in the ThreadWithAttributes extra folders, or alternately in 
     * thread local fields. 
     */  
    public interface Handler extends AbstractEndpoint.Handler {  
        public SocketState process(SocketWrapper<Long> socket,  
                SocketStatus status);  
    }</span>  


 

④在Http11AprProtocol类中实现了AprEndpoint中的Handler接口，

 

[java] view plain copy

protected static class Http11ConnectionHandler  
           extends AbstractConnectionHandler<Long,Http11AprProcessor> implements Handler {  
         

并调用Http11AprProcessor类(该类实现了ActionHook回调接口)。
 

       @Override

 

[java] view plain copy

<span style="font-size:14px;">        protected Http11AprProcessor createProcessor() {  
            Http11AprProcessor processor = new Http11AprProcessor(</span>  
 

 

四、 通过Connector实现一个简单的Server。
通过以下这些步骤，你就可以建立一个很简单的服务器，除了用到tomcat的一些类之外，跟Tomcat没有任何关系。
首先，建立一个含有main()方法的java类，详细代码如下：
[java] view plain copy

package myConnector;  
  
import java.util.concurrent.BlockingQueue;  
import java.util.concurrent.LinkedBlockingQueue;  
import java.util.concurrent.ThreadPoolExecutor;  
import java.util.concurrent.TimeUnit;  
  
import org.apache.coyote.http11.Http11Protocol;  
  
  
/** 
 * 基于Http11Protocol实现一个简单的服务器 
 * @author bingduanLin 
 * 
 */  
public class MyServer {  
  
    /** 
     * @param args 
     */  
    public static void main(String[] args) throws Exception{  
        Http11Protocol hp = new Http11Protocol();  
        hp.setPort(9000);  
        ThreadPoolExecutor threadPoolExecutor = createThreadPoolExecutor();  
        threadPoolExecutor.prestartCoreThread();  
        hp.setExecutor(threadPoolExecutor);  
        hp.setAdapter(new myHandler());  
        hp.init();  
        hp.start();  
        System.out.println("My Server has started successfully!");  
          
  
    }  
      
    public static ThreadPoolExecutor createThreadPoolExecutor() {  
        int corePoolSite = 2 ;  
        int maxPoolSite = 10 ;  
        long keepAliveTime = 60 ;  
        TimeUnit unit = TimeUnit.SECONDS;  
        BlockingQueue<Runnable> workQueue = new LinkedBlockingQueue<Runnable>();  
        ThreadPoolExecutor threadPoolExecutor =   
                new ThreadPoolExecutor(corePoolSite, maxPoolSite,  
                        keepAliveTime, unit, workQueue);  
        return threadPoolExecutor;  
    }  
  
}  

上面这个类用到的MyHandler类代码如下：
[java] view plain copy

package myConnector;  
  
import java.io.ByteArrayOutputStream;  
import java.io.OutputStreamWriter;  
import java.io.PrintWriter;  
  
import org.apache.coyote.Adapter;  
import org.apache.coyote.Request;  
import org.apache.coyote.Response;  
import org.apache.tomcat.util.buf.ByteChunk;  
import org.apache.tomcat.util.net.SocketStatus;  
  
public class myHandler implements Adapter {  
  
    @Override  
    public void service(Request req, Response res) throws Exception {  
        // 请求处理  
        System.out.println("Hi, Boss. I am handling the reuqest!");  
        ByteArrayOutputStream baos = new ByteArrayOutputStream();     
        PrintWriter writer = new PrintWriter(new OutputStreamWriter(baos));     
        writer.println("Not Hello World");     
        writer.flush();     
    
        ByteChunk byteChunk = new ByteChunk();     
        byteChunk.append(baos.toByteArray(), 0, baos.size());     
        res.doWrite(byteChunk);    
  
    }  
  
    @Override  
    public boolean event(Request req, Response res, SocketStatus status)  
            throws Exception {  
        System.out.println("Event-Event");  
        return false;  
    }  
  
    @Override  
    public boolean asyncDispatch(Request req, Response res, SocketStatus status)  
            throws Exception {  
        // TODO Auto-generated method stub  
        return false;  
    }  
  
    @Override  
    public void log(Request req, Response res, long time) {  
        // TODO Auto-generated method stub  
  
    }  
  
    @Override  
    public String getDomain() {  
        // TODO Auto-generated method stub  
        return null;  
    }  
  
}  

注意，必须导入相关的tomcat源码文件。完成之后运行主程序即：MyServer。然后在浏览器中输入：localhost:9000就可以看到“Not Hello World”。
下面是主程序的一些输出：
十二月 20, 2012 8:46:12 下午 org.apache.coyote.AbstractProtocol init
INFO: Initializing ProtocolHandler ["http-bio-9000"]
十二月 20, 2012 8:46:13 下午 org.apache.coyote.AbstractProtocol start
INFO: Starting ProtocolHandler ["http-bio-9000"]
My Server has started successfully!
Hi, Boss. I am handling the reuqest!
Hi, Boss. I am handling the reuqest!
Hi, Boss. I am handling the reuqest!
Hi, Boss. I am handling the reuqest!