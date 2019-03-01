---
layout: post
title: 无状态思想
tags:
- springboot
categories: thinking
description: 无状态思想
---
## 无状态
给定一个函数（方法）y=f(x)，对于一个给定的x，函数经过计算总是能得到相同的结果，不依赖外部的状态，那这个函数就是无状态的。  
好处：因为函数的内部变量和数据在使用完之后就释放了，不存在对外部数据的操作，在多线程的并发上不会存任何问题。


<!-- more -->

## http的无状态思想
HTTP每次访问一个静态HTML页面的时候，对于服务器来讲，就相当于调用了一个函数，**函数输入：一个URL路径， 函数输出：HTML页面**。这个服务器也不会记录每次请求的是谁，只要执行这个'函数调用'就可以了。  
好处：如果一个服务器访问量过大，可以轻松实现集群。（不必担心集群过程中用户状态不一致）  
问题：在服务器端需要保存用户状态，一个人登录了，我们得记住他是谁，他往购物车里加入商品，我们也得记下来。  
## 解决方案
1. 把状态（session）转移存储到另外一个地方，尽量服务器恢复到无状态的'y=f(x)'。  
![session转移](\assets\img\Stateless_1.jpg)
1. 服务器端不做session的维护，用户登陆时服务器取出用户身份标识先用摘要算法生成摘要信息，再用密钥将摘要加密生成签名，把签名发送给用户进行保管，当客户再次发出请求时携带上自己的身份标识和加密后的签名，服务器再用密钥对用户身份进行加密对比签名，然后认证用户身份。
![签名加密](\assets\img\Stateless_2.jpg)
## java无状态对象
无状态对象：一个对象没有实例变量，或者实例变量是final的(即不可变的实例变量)。  
Controller, Service 默认都是单例，运行期只有一个实例，他们的方法应该是y=f(x)这样的无状态方法，轻易不要在里边放置共享的实例变量，要不然多线程并发操作（tomcat就是多线程处理请求）就可能出问题了。  
但是如果确实需要共享的变量就把他放入threalocal  
### threalocal
ThreadLoal是每个线程保存私有变量的地方，因为多线程并发时操作公共变量容易导致变量被覆盖，所以可将这个公共变量存放在被使用线程的ThreadLoal中。  
用例：拦截器拦截每一个用户信息，将用户的user-agent信息存放在当前线程的ThreadLoca中，在controller的层层调用中的某一层可能会用到用户的user-agent信息，然后直接获取就行，**因为tomcat使用的是线程池处理用户请求，最后一定要删除存放的信息（原因是线程池的线程用完不会销毁，会变成阻塞状态等待下一个用户请求，里面存放的变量不会被删除）。**  
#### 代码
LocalUser：获取和设置ThreadLocal  
```
	package com.example.demo.util;

	/**
	 * Created by Administrator on 2018/11/26 0026.
	 */
	public class LocalUser {
		//处理每个线程的用户
		private static ThreadLocal<String> USER_LOCAL = new ThreadLocal<>();

		public static void setCurrentUser(String user) {
			USER_LOCAL.set(user);
		}
		public static String getCurrentUser(){
		  return  USER_LOCAL.get();
		}

		public static void removeUser(){
			USER_LOCAL.remove();
		}
	}
```
UserInterceptor：拦截器  
```
	package com.example.demo.interceptor;
	import com.example.demo.util.LocalUser;
	import org.springframework.stereotype.Component;
	import org.springframework.web.servlet.ModelAndView;
	import org.springframework.web.servlet.handler.HandlerInterceptorAdapter;

	import javax.servlet.http.HttpServletRequest;
	import javax.servlet.http.HttpServletResponse;

	/**
	 * Created by Administrator on 2018/11/26 0026.
	 */
	@Component
	public class UserInterceptor extends HandlerInterceptorAdapter {
		@Override
		public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
			String user = request.getHeader("User-Agent");
			LocalUser.setCurrentUser("i'm " + user);
			return true;
		}

		@Override
		public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex)
				throws Exception {
			LocalUser.removeUser();
		}
	}
```
Controller  
```
	package com.example.demo.controller;

	import com.example.demo.service.ShowUserAgentService;
	import com.example.demo.util.LocalUser;
	import org.springframework.beans.factory.annotation.Autowired;
	import org.springframework.web.bind.annotation.RequestMapping;
	import org.springframework.web.bind.annotation.RestController;

	/**
	 * Created by Administrator on 2018/11/26 0026.
	 */
	@RestController
	public class ShowUserController {
		@Autowired
		ShowUserAgentService showUserAgentService;
		@RequestMapping("showUser")
		public String showUser(){
			String user = showUserAgentService.getUserAgent();
			return user;
		}
	}
```
Service  
```
	package com.example.demo.service.serviceImpl;

	import com.example.demo.service.ShowUserAgentService;
	import com.example.demo.util.LocalUser;
	import org.springframework.stereotype.Service;

	/**
	 * Created by Administrator on 2018/11/26 0026.
	 */
	@Service
	public class ShowUserAgentServiceImpl implements ShowUserAgentService {
		@Override
		public String getUserAgent() {
			String userAgent = LocalUser.getCurrentUser();
			return userAgent;
		}
	}
```








