---
layout: post
title: 深入解析tomcat和jetty
tags:
- designPattern
categories: thinking
description:  深入解析tomcat和jetty
---
## 学习目的  
通过Web容器Tomcat和Jetty的学习掌握一些java设计模式的经典应用和优秀的系统架构思路。  

<!-- more -->

## servlet  
为了将http服务器的功能代码和我们需要编写的业务代码进行解耦，java专门定义了一个servlet接口。实现了servlet接口的业务类就叫servlet。  
但是http服务器如果管理这些servlet的生命周期并且负责调用这些servlet,又会和http服务器的功能耦合，于是又发明了Servlet容器来加载和管理这些业务类。HTTP服务器不直接跟业务类打交道，而是把请求交给servlet容器，Servlet 容器会将请求转发到具体的 Servlet，如果这个servlet还没有创建，就会加载这个servlet，并且调用它的接口方法。  
![tomcat](\assets\img\tomcat&jetty_1.jpg)  
servlet接口和servlet容器有一整套开发规范。tomcat和jetty都按照规范实现了tomcat容器。而我们的工作就是实现对应的servlet，将它注册到tomcat中的servlet中，其他工作由http服务器和servlet容器完成。  
### servlet接口  
servlet接口定义了五个方法：  
```
public interface Servlet {
    void init(ServletConfig config) throws ServletException;
    
    ServletConfig getServletConfig();
    
    void service(ServletRequest req, ServletResponse res）throws ServletException, IOException;
    
    String getServletInfo();
    
    void destroy();
}
```  
其中service方法是我们需要实现的具体业务代码。而ServletRequest和ServletResponse则是对http协议的封装，包括请求和响应信息。  
servlet在初始化时会调用init方法，在销毁的时候会调用destory方法。  
在配置web.xml时可以给servlet设置一些参数，ServletConfig类是在，并且通过getServletConfig拿到这些参数。  
genericServlet是Servlet的抽象实现，大部分servlet是在http环境中处理的。所以servlet规范还提供了httpServlet来继承genericServlet，如此我们只需要将业务代码写入doGet和doPost方法  
### servlet容器  
根据servlet规范，web应用需要有一定的目录结构，结构如下：  
```
| -  MyWebApp
      | -  WEB-INF/web.xml        -- 配置文件，用来配置 Servlet 等
      | -  WEB-INF/lib/           -- 存放 Web 应用所需各种 JAR 包
      | -  WEB-INF/classes/       -- 存放你的应用类，比如 Servlet 类
      | -  META-INF/              -- 目录存放工程的一些信息
```  
servlet规范中定义了servletContext这个接口来对应一个web应用。web应用部署好后servlet容器在加载时会初始化web应用并为每个一个用创建唯一的servletContext对象，一个web应用中的所有servlet通过servletContext对象共享数据，这些数据包括web应用初始化参数，文件资源等。通过它还可以实现servlet的转发。  
### 扩展机制  
由于servlet规范是大家都要遵守的，当这个规范不能满足业务的个性化需求时，就需要考虑它的扩展性，servlet提供了两种扩展机制：filter和listener  
filter是过滤器。这个接口允许你对请求和响应做统一化定制处理。Web应用部署完成后Servlet 容器需要实例化 Filter并把所有filter串成一个filterChain。当请求进来时获取第一个filter并调用doFilter方法。doFilter方法调用filterChain的下一个filter。  
listener是监听器。servlet容器在运行过程中会发生各种事件，如启动，停止，请求到达。servlet容器提供了一些默认的监听器来监听这些事件，当事件发生时会调用对应监听器的处理方法。也可以自定义一些监听器类，配置到web.xml监听需要的事件。  
## tomcat系统架构  
tomcat要解决的两个核心问题：  
- http服务器：处理socket连接，网络协议和Request 和 Response 对象的相互转化。  
- servlet容器：管理servlet,处理request请求。  
tomcat设计了两个核心组件解决这个问题：连接器（connector）负责对完交流,容器（container）负责管理内部。  
**tomcat支持的I/O模型：**  
IO：非阻塞 I/O，采用 Java NIO类库实现。  
NIO2：异步 I/O，采用 JDK 7 最新的 NIO2类库实现。  
APR：采用 Apache 可移植运行库实现，是 C/C++编写的本地库。  
**Tomcat 支持的应用层协议有：**  
HTTP/1.1：这是大部分 Web 应用采用的访问协议。  
AJP：用于和 Web 服务器集成（如 Apache）。  
HTTP/2：HTTP 2.0 大幅度的提升了 Web 性能。  
为了支持多种I/O模型和应用层协议，一个容器可以对应多个连接器。容器和连接器的组合叫做service。service才能对外提供一个完整的服务。同时一个tomcat中可能有多个service。  
![tomcat系统架构](\assets\img\tomcat&jetty_2.jpg)
### 连接器（connector）  
为了将网络字节流转化为ServletRequest和ServerResponse对象。连接器需要实现如下功能：  
- 监听网络端口。  
- 接受网络请求  
- 读取字节流请求   
- 根据应用层协议解析请求并生成Tomcat Request对象  
- 将Tomcat Request转成ServletRequest对象  
- 调用servlet容器，生成ServletResponse对象  
- 将 ServletResponse转成Tomcat Response对象  
- 将Tomcat Response对象转成网络字节流  
- 将字节流回写给浏览器  

通过分析，连接器有三个高内聚的功能：  
- 网络通信  
- 应用层协议解析  
- Tomcat Request/Response对象和Servlet Request/Response转换  

**每个高内聚的功能应该被封装成一个模块，模块之间通过抽象接口进行交互，降低模块之间的依赖性，一个模块的改动不影响其他模块。提高模块的复用性。**  
每个模块的内部功能可能时随时变动的，比如通信协议，对象转换，但是整体流程不变。  
通信模块给应用层模块提供字节流，应用层协议解析模块给对象转换模块提供Tomcat Request/Response对象，对象转换模块给servlet提供Servlet Request/Response对象。  
因此Tomcat设计了3 个组件来实现这3个功能。分别是EndPoint（通信组件），Processor（应用层协议解析组件）和Adapter（对象转换模块）  

由于I/O模型和应用层协议可以相互组合，比如NIO+HTTP或者IO+HTTP2，tomcat将网络通信和应用层协议的解析放在了一起，设计了一个叫ProtocolHandler的接口封装两种变化，各种协议和通信模型的组合的解析有相应的实现类。如Http11NioProtocol 和 AjpNioProtocol.   
![ProtocolHandler继承结构](\assets\img\tomcat&jetty_3.jpg)  
**尽量将稳定的部分放到抽象基类也是必不可少的设计原则**    
下面我们看看ProtocolHandler组件的具体设计  
**EndPoint**：是用来实现TCP/IP协议的。是一个接口，对应的抽象实现类是AbstractEndpoint，而在它的具体子类如NioEndPoint和Nio2EndPoint，有两个重要的子组件：Acceptor和SocketProcessor。  
其中Acceptor用来监听Socket的连接请求，SocketProcessor用来处理这些请求。SocketProcessor被提交到线程池里进行处理，这个线程池叫执行器（Executor）。  
**Processor**：用来解析应用层协议，并将其封装到Tomcat Request/Response。  
**Adapter**：tomcat定义的Tomcat Request/Response并不是标准的Servlet Request/Response。为了解决这个问题引入了CoyoteAdapter，这是适配器模式的经典运用
。连接器调用CoyoteAdapter的service方法，传入的是Tomcat Request，然后将其转化为Servlet Request再调用容器的service方法。  
![连接器组件图](\assets\img\tomcat&jetty_4.jpg)  
### 容器（container）  
tomcat设计了四种容器Engine、Host、Context 和 Wrapper。他们之间是父子关系，具有层次结构  
![容器](\assets\img\tomcat&jetty_5.jpg)  
其中一个wrapper代表一个servlet。context代表一个web应用，host代表一个站点，tomcat可以设置多个站点，一个host可以部署多个应用。一个engine代表一个引擎，可以管理多个站点。   
service是顶层组件，将连接器组件和容器组件封装到一起对外提供服务，一个service可以包含多个连接器，但是只能包含一个engine。  
因为各种不同容器之间具有树形的继承关系，为了实现对一个最上层容器Engine（整体，Engine包含了所有容器）的操作和对一个最底层的容器Wrapper（个体）对象的操作具有一致性，这里使用了组合模式。  
组合模式的实现方式：  
所有容器都实现了Container接口：  
```
public interface Container extends Lifecycle {
    public void setName(String name);
    public Container getParent();
    public void setParent(Container container);
    public void addChild(Container child);
    public void removeChild(Container child);
    public Container findChild(String name);
}
```
接口通过setParent和addChild来放置父容器和子容器。  
继承的Lifecycle管理各个容器的生命周期。  
**mapper**  
为了保证能在多层的容器中找到对应的wrapper容器中的servlet，通过mapper组件来处理容器容器组件和访问路径的映射关系。  
mapper维护了各个容器和路径的映射关系（分层次的map），mapper根据请求路径和容器的等级解析到对应的容器。  
**容器对请求的处理**  
并不是说只有servlet才会对请求进行处理，每层容器都会对请求进行处理，这个处理主要体现在Pipeline-Valve上。  
Pipeline-Valve是责任链模式，有多个处理者对这个请求依次进行处理。  
其中Value是每一个处理节点。通过setNext将Value串成一个处理链。  
```
public interface Valve {
  public Valve getNext();
  public void setNext(Valve valve);
  public void invoke(Request request, Response response)
}
```  
这个处理链维护在Pipeline中。  
```
public interface Pipeline extends Contained {
  public void addValve(Valve valve);
  public Valve getBasic();
  public void setBasic(Valve valve);
  public Valve getFirst();
}
```  
每个容器都有一个Pipeline对象，不同容器的Pipeline为了保证链式触发Valve的invoke，设置了一个first属性和basic属性分别指向每个容器的头valve和尾valve。basic会调用子容器的valve。  
![责任链](\assets\img\tomcat&jetty_6.jpg)  
整个调用过程是由Adapter触发的：  
```
// Calling the container
connector.getService().getContainer().getPipeline().getFirst().invoke(request, response);
```
Wrapper 容器的最后一个 Valve 会创建一个Filter 链，并调用 doFilter() 方法，最终会调到 Servlet 的 service 方法。  
### 用到的设计模式  
**适配器模式**：为了解决连接器中tomcat Request/Response对象和容器中Servlet Request/Response对象的不兼容问题，通过适配器调用容器，适配器是两个不兼容接口的桥梁。  
**组合模式**：容器组件中包含四种不同的容器，不同容器之间具有树形的继承关系，通过组合模式实现对一个最上层容器Engine（也就是整体，因为Engine包含所有的不同容器）的操作和对一个最底层的容器Wrapper对象（也就是个体）的操作一致。  
**责任链模式**：通过责任链模式保证每个层次的容器对请求依次进行处理。  
## tomcat如何启动
### tomcat处理请求  
tomcat中一个请求的处理流程如下：  
![处理流程](\assets\img\tomcat&jetty_7.png)  
从中可以看出组件是有大小之分的，server管理service，service管理连接器和容器。  
并且组件之间的调用关系是外层组件调用内层组件，其中连接器是外层组件，调用内层容器中的业务代码。  
### 组件创建原则
1，子组件应该先被创建，再创建父组件，最后子组件注入到父组件之中。  
2，内层组件应该先被创建，再创建外层组件，内层组件应该注入外层组件之中。  
**问题：**  
按照这种先小后大，先内后外将会导致组件后期的扩展变得混乱，并且组件注入的时候容易遗漏。  
为了防止这种问题，需要一个接口统一管理每个组件的生命周期。  
### LifeCycle接口  
每个组件都会有创建，初始化，启动这几个过程，这几个过程之间的转换是不会变的，会变的是每个过程的具体实现。
tomcat将这些不变的点抽象出来形成一个接口，叫做LifeCycle接口，也就是生命周期。接口里有这几个方法，init()，start()，stop()，destory()，每个具体的组件会去实现这些方法。
**父子组件的管理**：  
为了预防子组件注入父组件时会有遗漏，在父组件的init()中会调用子组件的init()方法，同样在父组件的start()中会调用子组件的start()方法。这就是组合模式的使用（整体和单个的操作一致）。  
**可扩展性：**  
当一个组件的init()或者其他生命周期方法改变时，为了不改变原来组件类中init()方法中的代码（开闭原则：为了扩展类中的功能而不改变原类中的代码，通过增加新的类），可以把组件生命周期定义成一个个状态，把状态的改变定义成一个个事件，而事件是有监听器的，通过实现监听器来完成功能的扩展，这就是观察者模式。  
在LifeCycle接口接口里有两个方法，添加监听器和删除监听器，除此之外还有一个枚举变量表示组件有哪些状态，以及在什么状态会触发什么事件。这就是LifeCycle和LifeCycleState.  
![LifeCycle和LifeCycleState](\assets\img\tomcat&jetty_8.png)  
**重用性：**  
有了接口，就需要类实现接口，可能不同的组件会有不同的实现类，但是不同的实现类会有相同 的的一些实现逻辑，为此可以定义一个基类来实现相同的逻辑，然后让各个子类实现他


