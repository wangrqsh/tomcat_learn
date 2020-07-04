https://blog.csdn.net/wangyangzhizhou

## 整体架构

#### tomcat介绍

`开源的 Java Web 应用服务器，实现了 Java EE(Java Platform Enterprise Edition)的部 分技术规范，比如 Java Servlet、JavaServer Pages、Java Expression Language、Java WebSocket。`

#### 目录结构

bin   `bin目录主要是用来存放tomcat的脚本，如startup.sh , shutdown.sh`

conf `目录下是配置文件`

- catalina.policy: Tomcat安全策略文件,控制JVM相关权限,具体可以参考java. security.Permission
- catalina.properties : Tomcat Catalina行为控制配置文件,比如Common ClassLoader
- logging.properties  : Tomcat日志配置文件, JDK Logging
- **server.xml** : Tomcat Server配置文件
- GlobalNamingResources :全局JNDI资源
- context.xml :  全局Context配置文件
- tomcat-users.xml : Tomcat角色配置文件
- web.xml :  Servlet标准的web.xml部署文件, Tomcat默认实现部分配置入内:
  - org.apache.catalina.servlets.DefaultServlet
  - org.apache.jasper.servlet.JspServlet

lib  `公共类库`

logs  `tomcat在运行过程中产生的日志文件`

webapps  `用来存放应用程序，当tomcat启动时会去加载webapps目录下的应用程序`

work  `用来存放tomcat在运行时的编译后文件，例如JSP编译后的文件`

#### 配置文件及脚本

```shell
# /bin/startup.sh 启动
EXECUTABLE=catalina.sh
exec "$PRGDIR"/"$EXECUTABLE" start "$@"
# /bin/shutdown.sh 关闭
EXECUTABLE=catalina.sh
exec "$PRGDIR"/"$EXECUTABLE" stop "$@"
# 启动和关闭都是调用 catalina.sh 脚本
# /bin/catalina.sh 发现如下2行
org.apache.catalina.startup.Bootstrap "$@" start
org.apache.catalina.startup.Bootstrap "$@" stop
```

`org.apache.catalina.startup.Bootstrap类是入口类,内部含有main方法,可以以此查看源码`

```properties
#catalina.properties
#限制可以访问的包
package.access=sun.,org.apache.catalina.,org.apache.coyote.,org.apache.jasper.,org.apache.tomcat.
#common类加载器可以加载的lib资源,catalina.base与catalina.home是相同
common.loader="${catalina.base}/lib","${catalina.base}/lib/*.jar","${catalina.home}/lib","${catalina.home}/lib/*.jar"
server.loader=默认空,公用common.loader
shared.loader=默认空,公用common.loader
```

```xml
<!-- server.xml  -->
<Server port="8005" shutdown="SHUTDOWN">
  <Service name="Catalina">
    <Executor name="tomcatThreadPool"namePrefix="exec-my"prestartminSpareThreads="true"
              maxThread="200"maxThreads="500" minSpareThreads="8"maxIdleTime="10000"/>
    <Connector port="8080"protocol="HTTP/1.1"executor="tomcatThreadPool"connectionTimeout="20000"redirectPort="8443" />
    <Engine name="Catalina" defaultHost="localhost">
      <Host name="localhost" appBase="webapps" unpackWARs="true" autoDeploy="true">
        <!--<Context docBase="D:\myapp" path="/xxx"  reloadable="true" />-->
      </Host>
    </Engine>
  </Service>
</Server>
```

#### web应用部署

- 部署到webapps目录下,该目录下默认每个目录都是一个应用,可以在server.xml文件中用 <Host/>标签自定义目录位置

  ```xml
  <Host name="localhost" appBase="webapps" unpackWARs="true" autoDeploy="true">
  ```

- Context标签下配置,在server.xml文件中用该标签配置

  ```xml
  <Context docBase="D:\myapp" path="/xxx"  reloadable="true" />
  ```

​       `path:指定访问该Web应用的URL入口`

​       ` docBase:指定Web应用的文件路径，可以给定绝对路径，也可以给定相对于<Host>的appBase属性的相对路径。`

​       ` reloadable: 如果这个属性设为true，tomcat服务器在运行状态下会监视在WEB-INF/classes和WEB-INF/lib目录下class文件的改动，`

​       ` 如果监测到有class文件被更新的，服务器会自动重新加载Web应用。`

- 在$CATALINA_BASE/conf/[enginename]/[hostname]/ 目录下（默认conf/Catalina/localhost）创建xml文件，文件名就是contextPath

​       比如创建api.xml，path就是/api,   `注意`：想要根目录访问，文件名为`ROOT.xml`

```xml
  <Context docBase="D:\api"  reloadable="true" />
```

#### 官方架构图

![image-20200622011035737](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20200622011035737.png)



#### 启动过程

```java
org.apache.catalina.startup.Bootstrap#main()-> bootstrap.init(); daemon.load(args); daemon.start();
org.apache.catalina.startup.Catalina#load()-> digester.parse(server.xml); getServer().init(); # start()->getServer().start();
org.apache.catalina.core.StandardServer#startInternal()-> for:services[i].start(); initInternal():for:services[i].init();
org.apache.catalina.core.StandardService#startInternal() -> engine.start(); for:executor.start(); for:connector.start();
org.apache.catalina.core.StandardEngine#startInternal()->
           findChildren().for:executor.submit(new StartChild(children[i]));->FutureTask->StartChild.start()
           ((Lifecycle) pipeline).start();
           threadStart()->new Thread(new ContainerBackgroundProcessor()).start();
```



#### 核心组件

- Server  (org.apache.catalina.Server)

  是指整个 Tomcat 服务器，包含多组服务，负责管理和启动各个Service，同时监听 8005 端口发过来的 shutdown 命令，用于关闭整个容器;

  org.apache.catalina.core.StandardServer

  ![image-20200622234004748](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20200622234004748.png)



- Service (org.apache.catalina.Service)

  Tomcat封装的、对外提供完整的基于组件的web服务,含有Connectors，Container2个核心组件，以及多个功能组件，各个service之间是独立的，共享同一个JVM资源，每个service组件都包含了若干个用于接收客户端消息的connector组件和处理请求的Engine组件.

  service组件还包含若干个Executor组件，每个都是一个线程池，他可以为service内所有组件提供线程池执行任务.

  org.apache.catalina.core.StandardService

  ![image-20200622235157108](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20200622235157108.png)

- Connector

  Tomcat 与外部世界的连接器，监听固定端口接收外部请求，传递给 Container，并将Container 处理的结果返回给外部.

  org.apache.coyote.http11.Http11AprProtocol  // AprEndpoint

  org.apache.coyote.http11.Http11NioProtocol  // NioEndpoint

  org.apache.coyote.http11.Http11Nio2Protocol // Nio2Endpoint

  ![image-20200623005548945](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20200623005548945.png)

- Container

  Catalina，Servlet容器，内部有多层容器组成，用于管理 Servlet 生命周期，调用 servlet 相关方法。

  - **Engine** ： Servlet 的顶层容器，包含一个或多个 Host 子容器；
  - **Host**：虚拟主机，负责 web 应用的部署和 Context 的创建；
  - **Context**：Web 应用上下文，包含多个 Wrapper，负责 web 配置的解析、管理所有的 Web 资源；
  - **Wrapper**：最底层的容器，是对 Servlet 的封装，负责 Servlet 实例的创建、执行和销毁。



![image-20200623004915475](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20200623004915475.png)

`子容器启动过程 `org.apache.catalina.core.ContainerBase#startStopExecutor.submit(new StartChild(children[i]))

​                         FutureTask->StartChild.start()

##### Context  应用加载

`tomcat如何加载web项目`

- WEB-INF/web.xml   

- 零xml配置 

  `spi @HandlesTypes(WebApplicationInitializer.class)`

  ```java
  org.apache.catalina.core.StandardContext#startInternal()
  org.springframework.web.SpringServletContainerInitializer#onStartup()
  ContextConfig#webConfig()
  org.apache.catalina.startup.ContextConfig#configureContext()
  //<load-on-startup>1</load-on-startup> 
  org.apache.catalina.core.StandardContext#loadOnStartup()  //servlet 初始化  
  ```

##### Tomcat启动机制(外置和内嵌)

`Tomcat启动带动IoC容器启动的逻辑：`

![image-20200623012834521](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20200623012834521.png)

`Spring boot中Tomcat容器和IoC容器的启动顺序`

- war外置： Tomcat启动带动IoC容器启动

- 内嵌：  Ioc容器带动Tomcat启动  

```java
org.springframework.boot.web.embedded.tomcat.TomcatWebServer#start
org.springframework.boot.web.embedded.tomcat.TomcatServletWebServerFactory#getWebServer
org.springframework.context.support.AbstractApplicationContext#refresh
```



#### 其他组件

- **Loader**：封装了 Java ClassLoader，用于 Container 加载类文件； 
- **Session**：负责管理和创建 session，以及 Session 的持久化(可自定义)，支持 session 的集群。
- **Pipeline**：在容器中充当管道的作用，管道中可以设置各种 valve(阀门)，请求和响应在经由管道中各个阀门处理，提供了一种灵活可配置的处理请求和响应的机制。
- **JMX**：Java SE 中定义技术规范，是一个为应用程序、设备、系统等植入管理功能的框架，通过 JMX 可以远程监控 Tomcat 的运行状态；
- **Realm**：Tomcat 中为 web 应用程序提供访问认证和角色管理的机制；
- **Jasper**：Tomcat 的 Jsp 解析引擎，用于将 Jsp 转换成 Java 文件，并编译成 class 文件。 
- **Naming**：命名服务，JNDI， Java 命名和目录接口，是一组在 Java 应用中访问命名和目录服务的 API。命名服务将名称和对象联系起来，使得我们可以用名称访问对象，目录服务也是一种命名 服务，对象不但有名称，还有属性。Tomcat 中可以使用 JNDI 定义数据源、配置信息，用于开发 与部署的分离。

## 工作原理

#### 生命周期

Tomcat 为了方便管理组件和容器的生命周期，定义了从创建、启动、到停止、销毁共 12 中状态，tomcat 生命周期管理了内部状态变化的规则控制，组件和容器只需实现相应的生命周期 方法即可完成各生命周期内的操作(initInternal、startInternal、stopInternal、 destroyInternal)。

![image-20200623072556539](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20200623072556539.png)

Tomcat 的生命周期管理引入了事件机制，在组件或容器的生命周期状态发生变化时会通 知事件监听器，监听器通过判断事件的类型来进行相应的操作。事件监听器的添加可以在 server.xml 文件中进行配置。

Tomcat 各类容器的配置过程就是通过添加 listener 的方式来进行的，从而达到配置逻辑与 容器的解耦。

- EngineConfig：主要打印start和stop事件的debug日志

- HostConfig：主要处理部署应用，解析应用 META-INF/context.xml 并创建应用的 Context 

- ContextConfig：主要解析并合并 web.xml，扫描应用的各类 web 资源 (filter、servlet、listener)

  

#### 请求处理流程

容器的责任链模式

1. 请求被Connector组件接收， 创建Request和Response对象。
2. Connector将Request和Response交给Container,先通过Engine的pipeline组件流经内部的每个Valve。
3. 请求流转到Host的pipeline组件中， 并且经过内部Valve的过滤。
4. 请求流转到Context的pipeline组件中， 并且经过内部的Valve的过滤。
5. 请求流转到Wrapper的pipeline组件中， 并且经过内部的Valve的过滤。
6. Wrapper内部的WrapperValve创建FilterChain实例， 调用指定的Servlet实例处理请求。
7. 返回

```java
Poller.run
HttpServlet#service

doFilter:168, ApplicationFilterChain (org.apache.catalina.core) [2]
invoke:728, ApplicationDispatcher (org.apache.catalina.core)
processRequest:470, ApplicationDispatcher (org.apache.catalina.core)
doForward:395, ApplicationDispatcher (org.apache.catalina.core)
forward:316, ApplicationDispatcher (org.apache.catalina.core)
renderMergedOutputModel:168, InternalResourceView (org.springframework.web.servlet.view)
render:304, AbstractView (org.springframework.web.servlet.view)
render:1286, DispatcherServlet (org.springframework.web.servlet)
processDispatchResult:1041, DispatcherServlet (org.springframework.web.servlet)
doDispatch:984, DispatcherServlet (org.springframework.web.servlet)
doService:901, DispatcherServlet (org.springframework.web.servlet)
processRequest:970, FrameworkServlet (org.springframework.web.servlet)
doGet:861, FrameworkServlet (org.springframework.web.servlet)
service:635, HttpServlet (javax.servlet.http)
service:846, FrameworkServlet (org.springframework.web.servlet)
service:742, HttpServlet (javax.servlet.http)
internalDoFilter:231, ApplicationFilterChain (org.apache.catalina.core)
doFilter:166, ApplicationFilterChain (org.apache.catalina.core) [1]
invoke:200, StandardWrapperValve (org.apache.catalina.core)
invoke:96, StandardContextValve (org.apache.catalina.core)
invoke:493, AuthenticatorBase (org.apache.catalina.authenticator)
invoke:140, StandardHostValve (org.apache.catalina.core)
invoke:81, ErrorReportValve (org.apache.catalina.valves)
invoke:650, AbstractAccessLogValve (org.apache.catalina.valves)
invoke:87, StandardEngineValve (org.apache.catalina.core)
service:342, CoyoteAdapter (org.apache.catalina.connector)
service:800, Http11Processor (org.apache.coyote.http11)
process:66, AbstractProcessorLight (org.apache.coyote)
process:806, AbstractProtocol$ConnectionHandler (org.apache.coyote)
doRun:1504, NioEndpoint$SocketProcessor (org.apache.tomcat.util.net)
run:49, SocketProcessorBase (org.apache.tomcat.util.net)
runWorker:1149, ThreadPoolExecutor (java.util.concurrent)
run:624, ThreadPoolExecutor$Worker (java.util.concurrent)
run:61, TaskThread$WrappingRunnable (org.apache.tomcat.util.threads)
run:748, Thread (java.lang)
```

![image-20200623080658816](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20200623080658816.png)

## 类加载机制 

### 双亲委派模型

- **Bootstrap ClassLoader** ：启动类加载器,负责加载 Java 的核心类,它不是 java.lang.ClassLoader 的子类,而是由 JVM自身实现,null  c,c++实现的,加载jre/lib

- **Extension ClassLoader** ：扩展类加载器，扩展类加载器的加载路径是 JDK 目录下 jre/lib/ext 。扩展加载器的 #getParent() 方法返回 null ，实际上扩展类加载器的父类加载器是启动类加载器。

- **System ClassLoader** ：系统(应用)类加载器，它负责在 JVM 启动时加载来自 Java 命令的 -classpath 选项、java.class.path 系统属性或 CLASSPATH 环境变量所指定的 jar 包和类路径。程序可以通过 #getSystemClassLoader() 来获取系统类加载器。系统加载器的加载路径是程序运行的当前路径。

ClassLoader#loadClass(java.lang.String, boolean)

jvm如何确定一个class唯一性：  全类名（包名+类名）+ classLoader的id

![image-20200624005731857](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20200624005731857.png)



### 类加载过程

![image-20200624010016905](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20200624010016905.png)

#### 类加载器

Tomcat 拥有不同的自定义类加载器，以实现对各种资源库的控制。 Tomcat 主要用类加载器解决以下 4 个问题：

- 同一个 Web 服务器里，各个 Web 项目之间各自使用的 Java 类库要互相隔离。

- 同一个 Web 服务器里，各个 Web 项目之间可以提供共享的 Java 类库 。
- 为了使服务器不受 Web 项目的影响，应该使服务器的类库与应用程序的类库互相独立。
- 对于支持 JSP 的 Web 服务器，应该支持热插拔（HotSwap）功能 。  

Tomcat提供了四组目录供用户存放第三方类库：

- 放置在/common目录中：类库可被Tomcat和所有的 Web应用程序共同使用。
- 放置在/server目录中：类库可被Tomcat使用，对所有的Web应用程序都不可见。
- 放置在/shared目录中：类库可被所有的Web应用程序共同使用，但对 Tomcat自己不可见。
- 放置在/WebApp/WEB-INF目录中：类库仅仅可以被此Web应用程序使用，对 Tomcat和其他Web应用程序都不可见。

Tomcat自定义了多个类加载器，CommonClassLoader、CatalinaClassLoader、SharedClassLoader和WebappClassLoader则是Tomcat自己定义的类加载器，它们分别加载/common/*、/server/*、/shared/*和/WebApp/WEB-INF/*中的Java类库。**其中WebApp类加载器和Jsp类加载器通常会存在多个实例**，**每一个Web应用程序对应一个WebApp类加载器**，**每一个JSP文件对应一个Jsp类加载器**。

```
LauncherHelper
```



## 线程模型

#### Tomcat对IO模型支持

| IO模型              | 描述                                                         |
| ------------------- | ------------------------------------------------------------ |
| BIO （JIoEndpoint） | 同步阻塞式IO，即Tomcat使用传统的java.io进行操作。该模式下每个请求都会创建一个线程，对性能开销大，不适合高并发场景。优点是稳定，适合连接数目小且固定架构。 |
| NIO（NioEndpoint）  | 同步非阻塞式IO，jdk1.4 之后实现的新IO。该模式基于多路复用选择器监测连接状态再同步通知线程处理，从而达到非阻塞的目的。比传统BIO能更好的支持并发性能。Tomcat 8.0之后默认采用该模式 |
| AIO  (Nio2Endpoint) | 异步非阻塞式IO，jdk1.7后之支持 。与nio不同在于不需要多路复用选择器，而是请求处理线程执行完成进行回调调知，继续执行后续操作。Tomcat 8之后支持。 |
| APR（AprEndpoint）  | 全称是 Apache Portable Runtime/Apache可移植运行库)，是Apache HTTP服务器的支持库。可以简单地理解为，Tomcat将以JNI的形式调用Apache HTTP服务器的核心动态链接库来处理文件读取或网络传输操作。使用需要编译安装APR 库 |

通过修改server.xml中protocol配置来指定IO模型

```xml
<Connector  protocol="HTTP/1.1"> 
```

#### Tomcat7连接器比较

![image-20200627232022253](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20200627232022253.png)



Tomcat8连接器比较

![image-20200627232055220](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20200627232055220.png)



#### JIoEndpoint原理

原理图：https://www.processon.com/view/link/5d490d2ae4b07f95f42b2bb0

![image-20200627233529873](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20200627233529873.png)

#### NioEndpoint原理

![image-20200627233615113](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20200627233615113.png)

## 性能调优

#### Tomcat启动参数

一般生产环境中Tomcat 程序目录和部署目录分开的，只需要在启动时指定CATALINA_HOME 与  CATALINA_BASE 参数即可。

| **启动参数**      | **描述说明**                                                 |
| :---------------- | :----------------------------------------------------------- |
| **JAVA_OPTS**     | jvm 启动参数 , 设置内存  编码等 -Xms100m -Xmx200m -Dfile.encoding=UTF-8 |
| JAVA_HOME         | 指定jdk 目录，如果未设置从java 环境变量当中去找。            |
| **CATALINA_HOME** | Tomcat 程序根目录                                            |
| **CATALINA_BASE** | 应用部署目录，默认为$CATALINA_HOME                           |
| CATALINA_OUT      | 应用日志输出目录：默认$CATALINA_BASE/log                     |
| CATALINA_TMPDIR   | 应用临时目录：默认：$CATALINA_BASE/temp                      |

#### Tomcat并发相关参数

​    https://tomcat.apache.org/tomcat-8.5-doc/config/http.html

Connector属性

| **名称**                         | **描述**                                                     |
| :------------------------------- | :----------------------------------------------------------- |
| address                          | 对于具有多个IP地址的服务器，此属性指定将用于监听指定端口的地址。默认情况下，连接器将侦听所有本地地址 |
| compression                      | 是否使用HTTP/1.1 GZIP压缩来节省服务器带宽，默认off           |
| connectionTimeout                | 客户端发起连接到服务端接收为止，中间最大的等待时间           |
| connectionUploadTimeout          | 指定数据上传过程中使用的超时时间(以毫秒为单位)。只有将disableUploadTimeout设置为false时才会生效。 |
| disableUploadTimeout             | true 则使用connectionTimeout                                 |
| enableLookups                    | 如果需要调用request.getRemoteHost()来执行DNS查找以返回远程客户端的实际主机名，则将其设置为true。设置为false可以跳过DNS查找并以字符串形式返回IP地址(从而提高性能)。默认情况下，DNS查找是禁用的 |
| executorTerminationTimeoutMillis | 私有内部执行程序在继续停止连接器之前等待请求处理线程终止的时间。如果没有设置，默认值为5000(5秒)。 |
| **acceptCount**                  | 使用所有可能的请求处理线程时传入连接请求的最大队列长度。队列满时接收到的任何请求都将被拒绝。默认值是100。 |
| **maxConnections**               | 服务器在任何给定时间将接受和处理的最大连接数。当到达此数值时，服务器将接受(但不处理)另一个连接。此附加连接将被阻塞，直到正在处理的连接数量低于maxConnections，此时服务器将再次开始接受和处理新连接。注意，一旦达到了限制，操作系统仍然可以基于acceptCount设置接受连接。默认值因连接器类型而异。对于NIO和NIO2，缺省值是10000。 |
| **maxHttpHeaderSize**            | 请求和响应HTTP头的最大大小，默认8k                           |
| **maxThreads**                   | 可以处理的并发请求的最大数量,默认200 ，**如果使用了executor，将被忽略** |
| **minSpareThreads**              | 始终保持运行的最小线程数，默认10，**如果使用了executor，将被忽略** |

Executor属性

| **名称**                | **描述**                                                     |
| :---------------------- | :----------------------------------------------------------- |
| daemon                  | 是否是守护线程,默认true                                      |
| namePrefix              | 由执行程序创建的每个线程的名称前缀。单个线程的线程名将是namePrefix+threadNumber |
| **maxThreads**          | 池中活动线程的最大数量，默认为200                            |
| **minSpareThreads**     | 保持活动的线程的最小数量(空闲和活动)，默认为25               |
| maxIdleTime             | 一个空闲线程关闭前的毫秒数，除非活动线程数小于或等于minSpareThreads。默认值为60000(1分钟) |
| **maxQueueSize**        | 在拒绝可运行任务之前，可以排队等待执行的最大可运行任务数。默认值是Integer.MAX_VALUE |
| prestartminSpareThreads | minSpareThreads是否应该在启动执行程序时启动，默认值都是false |





## tomcat+netty

org.apache.catalina.startup.Bootstrap

```java
// 加载catalina的启动类，构造一个加载catalina的类加载器，保证catalina依赖的类及文件对应用程序不可见,操作于 Catalina
private static Bootstrap daemon = null;
private Object catalinaDaemon = null;
ClassLoader commonLoader = null;
ClassLoader catalinaLoader = null;
ClassLoader sharedLoader = null;

private ClassLoader createClassLoader(String name, ClassLoader parent){...}
private void initClassLoaders() {
    commonLoader = createClassLoader("common", null);
    catalinaLoader = createClassLoader("server", commonLoader);
    sharedLoader = createClassLoader("shared", commonLoader);
}
// init(args[])
public void init(){
    initClassLoaders();
    Class<?> startupClass = catalinaLoader.loadClass("org.apache.catalina.startup.Catalina");
    Object startupInstance = startupClass.getConstructor().newInstance();
    Method method = startupInstance.getClass().getMethod("setParentClassLoader", paramTypes);
    method.invoke(startupInstance, paramValues);
    // get Catalina Daemon.
    catalinaDaemon = startupInstance;
}
public void start() throws Exception {
    if( catalinaDaemon==null ) init();
    Method method = catalinaDaemon.getClass().getMethod("start", (Class [] )null);
    method.invoke(catalinaDaemon, (Object [])null);
}
public void stop() throws Exception {
    Method method = catalinaDaemon.getClass().getMethod("stop", (Class [] ) null);
    method.invoke(catalinaDaemon, (Object [] ) null);
}
//stopServer(args[])
public void stopServer()throws Exception {
    Method method =catalinaDaemon.getClass().getMethod("stopServer", (Class []) null);
    method.invoke(catalinaDaemon, (Object []) null);
}
private void load(String[] arguments) throws Exception {
    // Call the load() method
    Method method = catalinaDaemon.getClass().getMethod( "load";, paramTypes);
    method.invoke(catalinaDaemon, param);
}
public static void main(String args[]) {
    //初始化一个守护进程变量、加载类和相应参数
    if (daemon == null) {
        // Don't set daemon until init() has completed
        Bootstrap bootstrap = new Bootstrap();
        try { bootstrap.init(); } catch (Throwable t) { return; }
        daemon = bootstrap;
    } else {
        // When running as a service the call to stop will be on a new
        // thread so make sure the correct class loader is used to prevent
        // a range of class not found exceptions.
        Thread.currentThread().setContextClassLoader(daemon.catalinaLoader);
    }
    // 解析命令，并执行
    try {
        String command = "start";
        if (args.length > 0) {
            command = args[args.length - 1];
        }
        if (command.equals("startd")) {
            args[args.length - 1] = "start";
            daemon.load(args);
            daemon.start();
        } else if (command.equals("stopd")) {
            args[args.length - 1] = "stop";
            daemon.stop();
        } else if (command.equals("start")) {
            daemon.setAwait(true);
            daemon.load(args);
            daemon.start();
            if (null == daemon.getServer()) {
                System.exit(1);
            }
        } else if (command.equals("stop")) {
            daemon.stopServer(args);
        } else if (command.equals("configtest")) {
            daemon.load(args);
            if (null == daemon.getServer()) {
                System.exit(1);
            }
            System.exit(0);
        } else {
            log.warn("Bootstrap: command \"" + command + "\" does not exist.");
        }
    } catch (Throwable t) {
        // Unwrap the Exception for clearer error reporting ...
        System.exit(1);
    }
}
```

   org.apache.catalina.startup.Catalina

```java
// 处理开始,停止运行的命令, 操作于server
protected String configFile = "conf/server.xml";
protected ClassLoader parentClassLoader = Catalina.class.getClassLoader();
// The server component we are starting or stopping.
protected Server server = null;
protected Thread shutdownHook = null;
// Prevent duplicate loads.
protected boolean loaded = false;
// the main digester to parse server.xml
protected Digester createStartDigester() {
   //Digester#parse(InputSource)> 在类 ObjectCreateRule#begin 中， 反射生成 StandardServer ...
   digester.addObjectCreate("Server","org.apache.catalina.core.StandardServer","className");
}
// stopServer()
public void stopServer(String[] arguments) {
    Server s = getServer();
    if (s == null) {
        // Create and execute our Digester
        Digester digester = createStopDigester();
        File file = configFile();
        try (FileInputStream fis = new FileInputStream(file)) {
            InputSource is = new InputSource(file.toURI().toURL().toString());
            is.setByteStream(fis);
            digester.push(this);
            digester.parse(is);
        } catch (Exception e) { system.exit(1);}
    } else {
        // Server object already present. Must be running as a service
        try {
            s.stop();
            s.destroy();
        } catch (LifecycleException e) {
            log.error("Catalina.stop: ", e);
        }
        return;
    }
    // Stop the existing server
    s = getServer();
    if (s.getPort()>0) {
        try (Socket socket = new Socket(s.getAddress(), s.getPort());
                OutputStream stream = socket.getOutputStream()) {
            String shutdown = s.getShutdown();
            for (int i = 0; i < shutdown.length(); i++) {
                stream.write(shutdown.charAt(i));
            }
            stream.flush();
        } catch (Exception ce) {
            System.exit(1);
        }
    } else {
        System.exit(1);
    }
}
// Start a new server instance.
// load(String args[])
public void load() {
    if (loaded) {
        return;
    }
    loaded = true;
    long t1 = System.nanoTime();
    //String temp = System.getProperty("java.io.tmpdir");
    initDirs();
    // Before digester - it may be needed
    initNaming();// Setting additional variables
    // Create and execute our Digester   -->解析server.xml
    Digester digester = createStartDigester();
    InputSource inputSource = null;
    InputStream inputStream = null;
    File file = null;
    try {
        try {
            file = configFile();  //  conf/server.xml
            inputStream = new FileInputStream(file);
            inputSource = new InputSource(file.toURI().toURL().toString());
        } catch (Exception e) {//fail}
        if (inputStream == null) {
            try {
                inputStream = getClass().getClassLoader()
                    .getResourceAsStream(getConfigFile());
                inputSource = new InputSource
                    (getClass().getClassLoader()
                     .getResource(getConfigFile()).toString());
            } catch (Exception e) {//fail}
        }
        // This should be included in catalina.jar
        // Alternative: don't bother with xml, just create it manually.
        if (inputStream == null) {
            try {
                inputStream = getClass().getClassLoader()
                    .getResourceAsStream("server-embed.xml");
                inputSource = new InputSource
                    (getClass().getClassLoader()
                     .getResource("server-embed.xml").toString());
            } catch (Exception e) {//fail }
        }
        if (inputStream == null || inputSource == null) {
            return;
        }
        try {
            inputSource.setByteStream(inputStream);
            digester.push(this);
            //解析xml配置
            digester.parse(inputSource);
            // org.apache.tomcat.util.digester.ObjectCreateRule 
            //反射创建Server
        } catch (Exception e) {
            return;
        }
    } finally {
        if (inputStream != null) {
            try {
                inputStream.close();
            } catch (IOException e) {
                // Ignore
            }
        }
    }

    getServer().setCatalina(this);
    getServer().setCatalinaHome(Bootstrap.getCatalinaHomeFile());
    getServer().setCatalinaBase(Bootstrap.getCatalinaBaseFile());

    // Stream redirection
    initStreams();

    // Start the new server
    try {
        getServer().init();
    } catch (LifecycleException e) {
        if (Boolean.getBoolean("org.apache.catalina.startup.EXIT_ON_INIT_FAILURE")) {
            throw new java.lang.Error(e);
        } else {
            log.error("Catalina.start", e);
        }
    }
  //  Initialization processed
}
    protected void initStreams() {
        // Replace System.out and System.err with a custom PrintStream
        System.setOut(new SystemLogHandler(System.out));
        System.setErr(new SystemLogHandler(System.err));
    }
// Start a new server instance.
    public void start() {
        if (getServer() == null) {
            load();
        }
        if (getServer() == null) {
            return;
        }
        // Start the new server
        try {
            getServer().start();
        } catch (LifecycleException e) {
            try {
                getServer().destroy();
            } catch (LifecycleException e1) {}
            return;
        }
        //Server startup in
        // Register shutdown hook
        if (useShutdownHook) {
            if (shutdownHook == null) {
                shutdownHook = new CatalinaShutdownHook();
            }
            Runtime.getRuntime().addShutdownHook(shutdownHook);
            // If JULI is being used, disable JULI's shutdown hook since
            // shutdown hooks run in parallel and log messages may be lost
            // if JULI's hook completes before the CatalinaShutdownHook()
            LogManager logManager = LogManager.getLogManager();
            if (logManager instanceof ClassLoaderLogManager) {
                ((ClassLoaderLogManager) logManager).setUseShutdownHook(false);
            }
        }

        if (await) {
            await();
            stop();
        }
    }    

    // Await and shutdown.
    public void await() {
       getServer().await();
    } 
    // Stop an existing server instance.
    public void stop() {
        try {
            // Remove the ShutdownHook first so that server.stop()
            // doesn't get invoked twice
            if (useShutdownHook) {
                Runtime.getRuntime().removeShutdownHook(shutdownHook);
                // If JULI is being used, re-enable JULI's shutdown to ensure
                // log messages are not lost
                LogManager logManager = LogManager.getLogManager();
                if (logManager instanceof ClassLoaderLogManager) {
                    ((ClassLoaderLogManager) logManager).setUseShutdownHook(true);
                }
            }
        } catch (Throwable t) {
            ExceptionUtils.handleThrowable(t);
            // This will fail on JDK 1.2. Ignoring, as Tomcat can run
            // fine without the shutdown hook.
        }
        // Shut down the server
        try {
            Server s = getServer();
            LifecycleState state = s.getState();
            if (LifecycleState.STOPPING_PREP.compareTo(state) <= 0
                    && LifecycleState.DESTROYED.compareTo(state) >= 0) {
                // Nothing to do. stop() was already called
            } else {
                s.stop();
                s.destroy();
            }
        } catch (LifecycleException e) {
            log.error("Catalina.stop", e);
        }
    }            
```

​       org.apache.tomcat.util.digester.Digester  => { 

​                           config(); //读取文件

​                           parse();//解析文件

​                           startElement();                             

​                                 //反射实例化对象,注册存储  

​                                 begin()-> digester.push(instance);

​                           endElement();

​                                 end();  -> 执行反射方法

​                       }





​      org.apache.catalina.Server

​      org.apache.catalina.util.LifecycleBase

​            org.apache.catalina.core.StandardServer

​                                startInternal() => service.start() => StandardService.startInternal () {

​                                              engine.start();

​                                            foreach => executor.start();

​                                             mapperListener.start()

​                                            foreach =>  connector.start();

​                                   }

![image-20200616081457080](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20200616081457080.png)

​           org.apache.catalina.core.StandardService

​                   org.apache.catalina.Engine

​                   org.apache.catalina.core.StandardEngine

​                   org.apache.catalina.mapper.MapperListener

​                   org.apache.catalina.connector.Connector

![image-20200616083622064](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20200616083622064.png)

![image-20200616083704849](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20200616083704849.png)



org.apache.catalina.connector.Connector ->  startInternal()

​            org.apache.coyote.ProtocolHandler -> protocolHandler.start()

​           org.apache.coyote.AbstractProtocol ->  start()

​           org.apache.tomcat.util.net.NioEndpoint ->  endpoint.start();

​                  ->  { bind();

​                           startInternal(); -> {  org.apache.tomcat.util.net.NioEndpoint.Poller  start}

​                           startAcceptorThreads();

​                       }

​                  org.apache.tomcat.util.net.NioSelectorPool

​                  org.apache.coyote.AbstractProtocol.ConnectionHandler  ->  process()



java.nio.channels.Selector

java.nio.channels.spi.AbstractSelector

sun.nio.ch.SelectorImpl

sun.nio.ch.WindowsSelectorImpl

org.apache.tomcat.util.net.NioEndpoint.PollerEvent

org.apache.tomcat.util.net.SocketWrapperBase

org.apache.tomcat.util.net.NioEndpoint.NioSocketWrapper





org.apache.catalina.core.ContainerBase

org.apache.catalina.core.StandardContext

org.apache.catalina.core.StandardHost

org.apache.catalina.core.StandardWrapper  ->  load();   loadServlet();   initServlet();   startInternal();    ....org.apache.tomcat.InstanceManager

![image-20200618074749952](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20200618074749952.png)

org.apache.catalina.Lifecycle





![image-20200618080440310](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20200618080440310.png)

org.apache.catalina.core.StandardEngineValve

org.apache.tomcat.util.digester.ObjectCreateRule

org.apache.tomcat.util.digester.SetPropertiesRule

org.apache.tomcat.util.modeler.modules.MbeansDescriptorsDigesterSource ->  execute()

org.apache.tomcat.util.modeler.Registry









