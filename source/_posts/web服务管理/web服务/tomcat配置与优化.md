---
title: tomcat配置与优化
date: 2019-06-06 17:26:08
tags:
    - tomcat
    - java
    - JVM
categories: ['web']
---

# JAVA程序
## java程序
* java程序设计语言
* java api【库函数】
* java class文件
* jvm：java虚拟机【字节码运行环境】
    - JRE：java运行时环境，只能运行class文件；
    - JDK：java开发环境，可将java程序编译为class文件；

![](https://simple0426-blog.oss-cn-beijing.aliyuncs.com/java-1.png)
## servlet
>Servlet类文件放到WEB-INF\classes目录下。

Servlet（小服务程序）是一个与协议无关的、跨平台的Web组件，它基于Java技术开发，由Servlet容器所管理。和运行在客户端浏览器中的Applet（小应用程序）相似，Servlet运行在服务器端。Servlet采用“请求—响应”模式提供Web服务，交互式地浏览和修改数据，生成动态Web内容。  
Servlet是平台独立的Java类，即按照Servlet规范编写的Java类，所以具有Java语言的所有优点，如良好的可移植性及安全性等。Servlet被编译为平台中立的字节码，可以被动态地加载到支持Java技术的Web服务器中运行，就如同Applet对客户端一样，区别在于Servlet运行并不需要图形用户界面。  
Java Servlet具有如下优点：  

* Servlet可以和其他资源（数据库、文件、Applet和Java应用程序等）交互，把生成的响应内容返回给客户端。如果需要，还可以保存“请求—响应”过程中的信息。
* 服务器采用Servlet可以完全授权对本地资源的访问，Servlet自身也会控制外部  用户的访问数量及访问性质。
* Servlet可以从本地硬盘，或者通过网络从远端硬盘来激活。
* 通过Servlet Tag技术（注：即HTML中标签），可以在HTML页面中动态调用Servlet。
* Servlet可以是其他服务的客户端程序。
* 通过链接技术，一个Servlet可以调用另一个或一系列Servlet来成为它的客户端。
* Servlet API与协议无关。

## jsp
是由Sun Microsystems公司倡导，在传统的网页HTML文件(*.htm,*.html)中插入Java程序段(Scriptlet)和JSP标记(tag)，从而形成JSP文件(*.jsp)。 用JSP开发的Web应用是跨平台的，既能在Linux下运行，也能在其他操作系统上运行。  
相应的ASP是微软公司倡导的。ASP是Active Server Page的缩写，意为“动态服务器页面”。ASP是微软公司开发的代替CGI脚本程序的一种应用，它可以与数据库和其它程序进行交互，是一种简单、方便的编程工具。ASP的网页文件的格式是 .asp。  
 jsp 要先翻译成servlet才能执行。JSP文件第一次执行时，要先由Tomcat将其转化为Servlet文件，然后编译，所以速度会慢一些，但后继执行时速度会很快；比如 test.jsp 要变成 test_jsp.java 然后编译成 test_jsp.class。而 test_jsp.java 本身就是一个servlet。所以 jsp只是servlet的一个变种，方便书写html内容才出现的。  
## war
在建立WAR文件之前，需要建立正确的Web应用程序的目录层次结构：

1. 建立WEB-INF子目录，并在该目录下建立classes与lib两个子目录。
2. 将Servlet类文件放到WEB-INF\classes目录下，将Web应用程序所使用Java类库文件（即JAR文件）放到WEB-INF\lib目录下。
3. 建立web.xml文件，放到WEB-INF目录下。
4. 根据Web应用程序的需求，将JSP页面或静态HTML页面放到上下文根路径下或其子目录下。
5. 如果有需要，建立META-INF目录

要注意的是，虽然WAR文件和JAR文件的文件格式是一样的，并且都是使用jar命令来创建，但就其应用来说，WAR文件和JAR文件是有根本区别的。JAR文件的目的是把类和相关的资源封装到压缩的归档文件中，而对于WAR文件来说，一个WAR文件代表了一个Web应用程序，它可以包含Servlet、HTML页面、Java类、图像文件，以及组成Web应用程序的其他资源，而不仅仅是类的归档文件。
# [tomcat][1]
与传统桌面应用程序不同，Tomcat中的应用程序是一个WAR（Web Archive）文件，它是许多文件构成的一个压缩包，包中的文件按照一定目录结构来组织，不同目录中的文件也具有不同的功能。部署应用程序时，只需要把WAR文件放到Tomcat的webapp目录下，Tomcat会自动检测和解压该文件。  
Tomcat既是一个Servlet容器，又是一个独立运行的服务器，像IIS、Apache等Web服务器一样，具有处理HTML页面的功能。但它处理静态HTML文件的能力并不是太强，所以一般都是把它当作JSP/Servlet引擎，通过适配器（Adapter）与其他Web服务器软件（如Apache）配合使用。此外，Tomcat还可与其他一些软件集成起来实现更多功能，例如，与JBoss集成起来开发EJB、与OpenJMS集成起来开发JMS应用、与Cocoon（Apache的另外一个项目）集成起来开发基于XML的应用等。  

## tomcat目录
### 主要目录
* bin：存放启动和关闭tomcat脚本
* conf：存放配置文件【server.xml、web.xml、logging.properties】
* lib：存放tomcat运行所需的库文件【jar】
* logs：存放tomcat运行时产生的日志
* webapps：tomcat的主要web程序发布目录
* work：存放jsp编译后产生的class文件

### 脚本设置
>catalina.sh变量设置；官方建议在bin目录下单独建立setenv.sh文件设置变量

* CATALINA_OUT：设置标准输出和错误输出，默认$CATALINA_BASE/logs/catalina.out
* CATALINA_OPTS：运行start、run、debug命令时的选项，但不被stop进程使用
* JAVA_HOME：设置jdk路径
* JAVA_OPTS：java运行时的选项，包含所有命令，当然也包含stop进程
* CATALINA_PID：设置进程pid，pid在强制停止【kill】进程时使用

### 脚本使用
- catalina.sh是控制tomcat启停的核心脚本
- startup/shutdown脚本都是调用catalina.sh的
- catalina.sh可选参数
    + run：控制台运行tomcat
    + start：后台运行tomcat【在另一个窗口运行tomcat】
    + stop [n]：关闭tomcat
        * -force：强制关闭tomcat【无法正常关闭时使用kill命令杀死进程】
    + configtest：对server.xml进行语法检查

## server.xml配置

```
<Server>
    <Service>
        <Connector/>
        <Engine>
            <Host>
                <Context>
                </Context>
            </Host>
         </Engine>
    </Service>
</Server>
```

### Server
一个server代表整个catalina servlet容器：

- port：指定Tomcat服务器监听shutdown命令的端口
- shutdown：指定终止tomcat服务器运行时，发给tomcat服务器shutdown监听端口的字符串

### Service
service是一个集合，它包含一个engine以及一个或多个connector组成

### Connector
一个connector将在某个端口监听客户请求，并把请求交给engine处理，同时从engine获取响应返回给客户端

- http连接器：监听来自客户端的http请求；
    + port：接受http请求的端口
    + redirectPort：如果端口不支持ssl时，将请求转发
    + connectionTimeout：请求超时时间【以毫秒计，默认20s，设置-1关闭超时】
    + maxHttpHeaderSize：请求和响应的http头部大小，以字节计数，【默认8192即8KB】
    + acceptCount：指定当所有可以使用的处理请求的线程数都被使用时，可以放到处理队列中的请求数，超过这个数的请求将不予处理【默认100】
    + maxConnections：任何时间都可以接收和处理的请求总数，超过此阈值但不超过acceptCount的请求依然可以接收【默认值10000】
    + minSpareThreads：tomcat保持的最小线程数【默认为10】
    + maxThreads：tomcat可以创建的最大线程数【默认200】
    + SSLEnabled：是否开启ssl【默认False】
- ajp连接器： 监听来自Webserver(apache)的servlet/jsp代理请求；<Connector port="8009" protocol="AJP/1.3" redirectPort="8443" />

### Engine
是一个service下的请求处理功能组件，可以包含多个虚拟主机(Host)

- defaultHost指定缺省的请求处理主机名，它至少与其中一个host元素的name属性值是一样

### Host
一个虚拟主机匹配一个域名，它可以包含多个应用(Context)

- name：主机名
- appBase：应用程序基本目录，可以指定绝对路径，也可以指定相对于CATALINA_HOME的相对路径
    + 这个目录下面的子目录自动被当成应用被部署
    + 这个目录下边的war文件自动被解压并作为应用部署
- unpackWARs：如果设置为true，则tomcat会自动解压war文件，否则直接运行war文件
- autoDeploy：设置为true时，当tomcat处于运行状态时，能够监测appBase下的文件，如果有新的wweb加入的话，会自动发布这个web应用

### Context
一个应用程序对应一个Context，一个web程序由一个或多个servlet组成

- docBase：【只有在应用不在appBase下才需要设置】应用程序的路径或者是war文件的路径
- path：表示此web应用的url前缀
- reloadable：如果设置为true，则tomcat会自动检测应用程序的WEB-INF/lib和WEB-INF/classes目录下class文件变化，自动重载新的应用程序，这样可以在不重启tomcat情况下改变应用程序

### Value
可以在（Engine, Host, or Context）级别设置不同功能的阈值，以下以设定访问日志为例

- className="org.apache.catalina.valves.AccessLogValve"：访问日志类名
- directory="logs"：日志存储目录
- prefix="localhost_access_log" ：日志文件前缀
- suffix=".txt"：日志文件后缀
- pattern="[%{yyyy-MM-dd HH:mm:ss}t] &quot;%r&quot; %S &quot;%{postdata}r&quot; %s %{Referer}i [%{User-Agent}i] %T %b"：日志格式

## 日志配置
* server.xml：配置访问日志【Value属性】
    - 需要在logrotate中配置日志轮询
* logging.properties：配置tomcat自身日志、console输出
    - 可以将1catalina、2localhost、3manager、4host-manager内容全部注释或将其prefix设置为同一个文件
* log4j.properties：项目自定义日志
    - 位置：项目WEB-INF/classes下

# JVM调优
按照存储数据的内容，将内存分配为堆区(heap)和非堆区(non-heap)

* 堆区：通过new的方式创建的对象(类实例)占用的内存空间，java的垃圾回收机制可以回收堆区占用的内存，调优参数如下
    * [-Xms n][2]：设置初始java堆【JVM】大小
        - 默认，JVM初始大小是物理内存的1/64，
        - 最小是8MB
    * [-Xmx n][3]：设置最大java堆【JVM】大小
        - 默认，JVM最大大小是物理内存的1/4
        - 【server模式下】最大值是32GB【此时内存大于等于128GB】
        - 【client模式下】最大值是256MB【大于等于1GB都被当做1GB处理】
* 非堆区：代码、常量、外部访问(文件访问占用的资源流)等，调优参数如下
        + -XX:PermSize=n：非堆区初始内存大小
        + -XX:MaxPermSize=n：非堆区最大可用内存

# 更多参考
* java性能调优书籍：《Java Performance: The Definitive Guide》
* [jvm设置][4]

[1]: http://tomcat.apache.org/tomcat-8.0-doc/config/index.html
[2]: https://segmentfault.com/q/1010000007235579?_ea=1280245
[3]: https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/parallel.html#default_heap_size
[4]: https://blog.csdn.net/losetowin/article/details/78569001
