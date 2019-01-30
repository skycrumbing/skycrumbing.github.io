---
layout: post
title: springCloud
tags:
- spring
- springCloud
categories: micro-services
description: 微服务架构的概念以及搭建。
---
## 微服务
微服务架构将传统的单体应用按照业务拆分成多个服务，每个服务是一个单独的进程，她们之间可以通过http协议进行通信。

<!-- more -->

## 微服务架构的优势与弊端
- 稳定性好，一个服务挂掉了不影响其他服务的运作（高内聚，低耦合）  
- 理解性强，不同业务的代码存放在不同的服务中而不是全部都在一个应用中，便于理解。  
- 可以对部分高访问量的服务实现集群。  
- 但是他也有不好的地方，比方说不方便调试，开销大（网络开销和服务器开销），还有就是开发难度高。  
## 相关概念
**分布式**：把一个业务划分为多个子业务，部署到不同的服务器上  
**集群**：把同一个业务部署到不同的服务器上  
**负载均衡**：当一个业务部署到不同服务器时（实现集群），用户请求一个业务处理该访问哪个服务器呢，这就需要负载均衡器实现负载均衡算法，拦截用户请求，通过负载均衡算法将请求转发到一台具体的服务器，服务器再响应请求返回给负载均衡器，负载均衡器再响应给用户  
**服务消费者**：使用服务的服务  
**服务生产者**：被使用的服务  
**断路器**：当一个业务在服务器挂掉时，用户请求它会会执行迅速失败的回调方法，而不是等待响应超时，不会造成请求阻塞  
**路由**：路由功能是微服务的一部分，比如将／api/user转发到user服务。  
## 搭建微服务架构  
### 服务注册和发现
项目采用Eureka作为服务注册与发现的组件  
- 首先创建一个主Maven工程，在其pom文件引入依赖，这个pom文件作为父pom文件，起到依赖版本控制的作用，本文其他module工程继承该pom。  
```
   <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.0.3.RELEASE</version>
        <relativePath/>
    </parent>
    <modules>
        <module>eureka-server</module>
        <module>service-hi</module>
        <module>service-ribbon</module>
        <module>service-feign</module>
        <module>service-zuul</module>
        <module>config-server</module>
        <module>config-client</module>
    </modules>
```
- 然后创建1个model作为服务注册中心，即eureka-server  
eureka-server的pom文件:  
```
	<parent>
		<groupId>SpringCloudDemo</groupId>
		<artifactId>skycrumb</artifactId>
		<version>1.0-SNAPSHOT</version>
	</parent>
	<dependencies>
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
		</dependency>

		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-web</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-config-server</artifactId>
		</dependency>
	</dependencies>
```
- 在eureka-serve的springboot启动类中添加@EnableEurekaServer注解启动服务注册中心，并增添appication.yml 配置文件   
```
	server:
	  port: 8889
	eureka:
	  instance:
	    hostname: localhost
	  client:
	    registerWithEureka: false
	    fetchRegistry: false
	    serviceUrl:
	      defaultZone: http://${eureka.instance.hostname}:${server.port}/eureka/
	spring:
	  application:
	    name: eurka-server
```
eureka.client.registerWithEureka：false和fetchRegistry：false来表明自己是一个eureka server  
- 启动eureka-server，打开http://localhost:8889，出现如下界面  
![界面显示](\assets\img\springcloud_1.jpg) 
- 服务注册中心搭建好了之后我们就需要创建一个服务提供者了，然后创建1个model作为服务提供者，即eureka-hi  
eureka-hi的pom文件  
```
	<parent>
		<groupId>SpringCloudDemo</groupId>
		<artifactId>skycrumb</artifactId>
		<version>1.0-SNAPSHOT</version>
	</parent>
	<dependencies>
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter-eureka</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-web</artifactId>
		</dependency>
	</dependencies>
```  
- 在eureka-hi的springboot启动类中添加@EnableEurekaClient 表明自己是一个eurekaclient并增添配置文件  
启动类如下  
```
	@SpringBootApplication
	@EnableEurekaClient
	@RestController
	public class ServiceHiApplication {

		public static void main(String[] args) {
			SpringApplication.run(ServiceHiApplication.class, args);
		}

		@Value("${server.port}")
		String port;

		@RequestMapping("/hi")
		public String home(@RequestParam(value = "name", defaultValue = "skycrumbing") String name) {
			return "hi " + name + " ,i am from port:" + port;
		}
	}
```  
配置文件如下（需要指明spring.application.name,这个很重要，这在以后的服务与服务之间相互调用一般都是根据这个name。）:  
```
	server:
	  port: 8762
	spring:
	  application:
	    name: service-hi
	eureka:
	  client:
	    serviceUrl:
	      defaultZone: http://localhost:8889/eureka/
```  
启动工程，打开http://localhost:8889，即可发现server-hi已经注册在服务注册中心  
![界面显示](\assets\img\springcloud_2.jpg)  
- 这时打开 http://localhost:8763/hi?name=skycrumbing ，你会在浏览器上看到  
hi skycrumbing ,i am from port:8763  
### 服务消费者与负载均衡
这里要用ribbon组件提供负载均衡的作用，当然也用到了Hystrix组件提供断路器作用。
还可以用 feign实现负载均衡的作用，就不在展示。  
- 创建1个model，service-ribbon并引入如下依赖  
```
	<parent>
		<groupId>SpringCloudDemo</groupId>
		<artifactId>skycrumb</artifactId>
		<version>1.0-SNAPSHOT</version>
	</parent>
	<dependencies>
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-web</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter-netflix-ribbon</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework</groupId>
			<artifactId>spring-web</artifactId>
			<version>RELEASE</version>
		</dependency>
		<!--断路器支持-->
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
		</dependency>
	</dependencies>
```  
配置文件如下：  
```
	eureka:
	  client:
	    serviceUrl:
	      defaultZone: http://localhost:8889/eureka/
	server:
	  port: 8764
	spring:
	  application:
	    name: service-ribbon
```
- 在工程的启动类中,通过@EnableDiscoveryClient向服务中心注册；并且向程序的ioc注入一个bean: restTemplate;并通过@LoadBalanced注解表明这个restRemplate开启负载均衡的功能。  
```
	//通过RestTemplate+Ribbon去消费服务
	@SpringBootApplication
	@EnableEurekaClient
	//消费者服务通过@EnableDiscoveryClient向服务中心注册
	@EnableDiscoveryClient
	//开启Hystrix
	@EnableHystrix
	public class ServiceRibbonApplication {

		public static void main(String[] args) {
			SpringApplication.run(ServiceRibbonApplication.class, args);
		}

		@Bean
	//	@LoadBalanced注解表明这个restRemplate开启负载均衡的功能。
		@LoadBalanced
		RestTemplate restTemplate() {
			return new RestTemplate();
		}
	}
```
- 写一个测试类HelloService，通过之前注入ioc容器的restTemplate来消费service-hi服务的“/hi”接口，在这里我们直接用的程序名替代了具体的url地址，在ribbon中它会根据服务名来选择具体的服务实例，根据服务实例在请求的时候会用具体的url替换掉服务名，代码如下：  
```
	@Service
	public class HelloService {
	//    通过之前注入ioc容器的restTemplate来消费service-hi服务的“/hi”接口
	    @Autowired
	    RestTemplate restTemplate;
	    //在hiService方法上加上@HystrixCommand注解。该注解对该方法创建了熔断器的功能，并指定了fallbackMethod熔断方法，熔断方法直接返回了一个字符串，字符串为”hi,”+name+”,sorry,error!”，
	    //启动：service-ribbon,service-hi工程，当我们访问http://localhost:8764/hi?name=forezp,浏览器显示hi forezp,i am from port:8762
	    //关闭 service-hi 工程，当我们再访问http://localhost:8764/hi?name=forezp，浏览器显示hi ,forezp,orry,error!
	    //这就说明当 service-hi 工程不可用的时候，service-ribbon调用 service-hi的API接口时，会执行快速失败，直接返回一组字符串，而不是等待响应超时，这很好的控制了容器的线程阻塞。
	    @HystrixCommand(fallbackMethod = "hiError")
	    public String hiService(String name) {
	//        在这里我们直接用的程序名替代了具体的url地址，在ribbon中它会根据服务名来选择具体的服务实例，根据服务实例在请求的时候会用具体的url替换掉服务名
		return restTemplate.getForObject("http://SERVICE-HI/hi?name="+name,String.class);
	    }

	    public String hiError(String name) {
		return "hi,"+name+",sorry,error!";
	    }
	}
```  
- 启动service-hi，eureka-server工程，它的端口为8763；将service-hi的配置文件的端口改为8762,并再次启动，再启动（在启动配置中将single instance only的勾选去掉），这时你会发现：service-hi在eureka-server注册了2个实例，这就相当于一个小的集群。
- 写一个controller，在controller中用调用HelloService 的方法，代码如下  
```  
	//  一个服务注册中心，eureka server,端口为8761 service-hi工程跑了两个实例，端口分别为8762,8763，
	//  分别向服务注册中心注册 sercvice-ribbon端口为8764,向服务注册中心注册 当sercvice-ribbon通过restTemplate
	//  调用service-hi的hi接口时，因为用ribbon进行了负载均衡，会轮流的调用service-hi：8762和8763 两个端口的hi接口；
	@RestController
	public class HelloControler {

		@Autowired
		HelloService helloService;
	//    @GetMapping是一个组合注解，是@RequestMapping(method = RequestMethod.GET)的缩写
		@GetMapping(value = "/hi")
		public String hi(@RequestParam String name) {
		    return helloService.hiService( name );
		}
	}
```
- 启动server-ribbon工程，访问http://localhost:8764/hi?name=skycrumbing  
 这时会轮流显示  
hi skycrumbing ,i am from port:8763  
hi skycrumbing ,i am from port:8762  
- 路由网关
这里使用了zuul组件完成路由的转发和过滤。  
- 创建1个model，service-zuul并引入如下依赖  
```
	<parent>
		<groupId>SpringCloudDemo</groupId>
		<artifactId>skycrumb</artifactId>
		<version>1.0-SNAPSHOT</version>
	</parent>

	<dependencies>
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-web</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter-netflix-zuul</artifactId>
		</dependency>

	</dependencies>
```  
- 在其入口applicaton类加上注解@EnableZuulProxy，开启zuul的功能  
```
	//Zuul的主要功能是路由转发和过滤器
	//zuul默认和Ribbon结合实现了负载均衡的功能
	@SpringBootApplication
	@EnableZuulProxy
	@EnableEurekaClient
	@EnableDiscoveryClient
	public class ServiceZuulApplication {

		public static void main(String[] args) {
			SpringApplication.run(ServiceZuulApplication.class, args);
		}
	}
```
- 添加配置文件application.yml  
```
	eureka:
	  client:
	    serviceUrl:
	      defaultZone: http://localhost:8889/eureka/
	server:
	  port: 8769
	spring:
	  application:
	    name: service-zuul
	zuul:
	  routes:
	    api-a:
	      path: /api-a/**
	      serviceId: service-ribbon
	    api-b:
	      path: /api-b/**
	      serviceId: service-feign
```
- 开启所有相关服务，访问http://localhost:8769/api-a/hi?name=skycrumbing  
会轮流显示：  
hi skycrumbing ,i am from port:8763  
hi skycrumbing ,i am from port:8762  
这说明zuul起到了路由的作用。  
同时zuul也可以实现过滤的作用，可以做安全验证。  
- 增加MyFilter类  
```
	//zuul不仅只是路由，并且还能过滤，做一些安全验证
	@Component
	public class MyFilter extends ZuulFilter {

	    private static Logger log = LoggerFactory.getLogger(MyFilter.class);
	   /* filterType：返回一个字符串代表过滤器的类型，在zuul中定义了四种不同生命周期的过滤器类型:pre：路由之前,routing：路由之时,post： 路由之后,error：发送错误调用
	    filterOrder：过滤的顺序
	    shouldFilter：这里可以写逻辑判断，是否要过滤，本文true,永远过滤。
	    run：过滤器的具体逻辑。可用很复杂，包括查sql，nosql去判断该请求到底有没有权限访问。*/
	    @Override
	    public String filterType() {
		return "pre";
	    }

	    @Override
	    public int filterOrder() {
		return 0;
	    }

	    @Override
	    public boolean shouldFilter() {
		return true;
	    }

	    @Override
	    public Object run() {
		RequestContext ctx = RequestContext.getCurrentContext();
		HttpServletRequest request = ctx.getRequest();
		log.info(String.format("%s >>> %s", request.getMethod(), request.getRequestURL().toString()));
		Object accessToken = request.getParameter("token");
		if(accessToken == null) {
		    log.warn("token is empty");
		    ctx.setSendZuulResponse(false);
		    ctx.setResponseStatusCode(401);
		    try {
			ctx.getResponse().getWriter().write("token is empty");
		    }catch (Exception e){}

		    return null;
		}
		log.info("ok");
		return null;
	    }
	}
```
- 这时访问：http://localhost:8769/api-a/hi?name=skycrumbing  
token is empty  
访问：http://localhost:8769/api-a/hi?name=skycrumbing&&token=22  
会轮流显示：
hi skycrumbing ,i am from port:8763  
hi skycrumbing ,i am from port:8762  
## 结尾
当然springcloud还涉及到分布式配置，消息总线，服务链追踪，断路器监控等，这些将在后续的学习中继续深入























