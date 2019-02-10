---
layout: post
title: springboot前后端分离
tags:
- - springboot
categories: spring
description: springboot前后端分离
---
## springboot
Spring Boot是由Pivotal团队提供的全新框架，其设计目的是用来**简化新Spring应用的初始搭建以及开发过程**。该框架使用了特定的方式来进行配置，从而使开发人员不再需要定义样板化的配置。

<!-- more -->

## restful
RESTful风格：一种软件架构风格。从MVC到前后端完全分离。首先从浏览器发送AJAX请求，然后服务端接受该请求并返回JSON数据返回给浏览器，最后在浏览器中进行界面渲染服务端将内部资源发布**REST服务**，客户端通过URL来定位这些资源并通过HTTP协议来访问它们。(前后端分离架构不一定是restful风格，但是restful风格一定是前后端分离的)
## Springboot如何实现前后端分离
主要改变在controller层，原先的controller与前端完成彻底分离，只是进行json数据的传输
- **注解的改变，@Controller需要变成RestController**  
使用@Controller 注解，在对应的方法上，视图解析器可以解析return 的jsp,html页面，并且跳转到相应页面 若返回json等内容到页面，则需要加@ResponseBody注解  
@RestController注解相当于@ResponseBody ＋ @Controller合在一起的作用。  
@RestController不能解析return 的jsp,html页面  
- **Json数据的封装**
除了封装要发送的数据外还要添加其他元数据，如HTTP状态信息表示服务器响应状态信息，要将他们封装到一个对象中
```
	@Data
	public class Meta {
		private boolean success;
		private String message;

		public Meta(boolean success, String message) {
			this.success = success;
			this.message = message;
		}

		public Meta() {
		}
	}
```
```
	@Data
	public class Response {
		public static final String OK = "ok";
		public static final String ERROR = "error";

		private Meta meta; //元数据
		private Object date; //具体数据

		public Response success(){
			this.meta = new Meta(true,OK);
			return this;
		}
		//重载
		public Response success(Object obj){
			this.meta = new Meta(true,OK);
			this.date = obj;
			return this;
		}
		public Response failure(){
			this.meta = new Meta(false,ERROR);
			return  this;
		}
		//重载
		public Response failure(Object obj){
			this.meta = new Meta(false,ERROR);
			this.date = obj;
			return  this;
		}
```
- **将元数据动态的添加进去，这就涉及Spring的aop了，增加一个处理异常的切面，他会在@requestMap注解的方法上的入口自动进行插入**
```
	@ControllerAdvice  //控制器增强,和@ExceptionHandler、@InitBinder、@ModelAttribute 等注解配合使用
	@ResponseBody //返回json
	public class ExceptionAspect {
		private static final Logger log = Logger.getLogger(ExceptionAspect.class);

		//400
		@ResponseStatus(HttpStatus.BAD_REQUEST)
		@ExceptionHandler(NoHandlerFoundException.class)
		public Response httpMessageNotReadableException(NoHandlerFoundException e){
			log.error("could_not_read_json...", e);
			return new Response().failure("could_not_read_json");
		}

		/**
		 * 400 - Bad Request
		 */
		@ResponseStatus(HttpStatus.BAD_REQUEST)
		@ExceptionHandler({MethodArgumentNotValidException.class})
		public Response handleValidationException(MethodArgumentNotValidException e) {
			log.error("parameter_validation_exception...", e);
			return new Response().failure("parameter_validation_exception");
		}

		/**
		 * 405 - Method Not Allowed。HttpRequestMethodNotSupportedException
		 * 是ServletException的子类,需要Servlet API支持
		 */
		@ResponseStatus(HttpStatus.METHOD_NOT_ALLOWED)
		@ExceptionHandler(HttpRequestMethodNotSupportedException.class)
		public Response handleHttpRequestMethodNotSupportedException(
				HttpRequestMethodNotSupportedException e) {
			log.error("request_method_not_supported...", e);
			return new Response().failure("request_method_not_supported");
		}

		/**
		 * 415 - Unsupported Media Type。HttpMediaTypeNotSupportedException
		 * 是ServletException的子类,需要Servlet API支持
		 */
		@ResponseStatus(HttpStatus.UNSUPPORTED_MEDIA_TYPE)
		@ExceptionHandler({ HttpMediaTypeNotSupportedException.class })
		public Response handleHttpMediaTypeNotSupportedException(Exception e) {
			log.error("content_type_not_supported...", e);
			return new Response().failure("content_type_not_supported");
		}

		/**
		 * 500 - Internal Server Error
		 */
		@ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
		@ExceptionHandler(Exception.class)
		public Response handleException(Exception e) {
			log.error("Internal Server Error...", e);
			return new Response().failure("Internal Server Error");
		}

	}
```
- **跨域**  
不同源的客户端脚本在没有明确授权的情况下，不能读写对方资源，简单来说就是不允许跨域的ajax请求。什么是跨域，URL，端口，协议，只要有一个不同就是跨域。要想进行跨域，必须在服务端进行授权   
授权方式：将CORS响应头写入response对象中即可  
所以要对http请求进行过滤操作，将所有请求的response对象中添加跨域响应头  
```
	@WebFilter(filterName="corsFilter",urlPatterns = "/*")
	public class CorsFilter implements Filter {
		private static final Logger log = Logger.getLogger(UserController.class);
		private String allowOrigin;
		private String allowMethods;
		private String allowCredentials;
		private String allowHeaders;
		private String exposeHeaders;
		@Override
		public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain) throws IOException, ServletException {
			HttpServletResponse httpServletResponse = (HttpServletResponse) servletResponse;
			httpServletResponse.setHeader("Access-Control-Allow-Origin", "*");//允许访问的客户端域名
			//允许访问的方法名,浏览器会向所请求的服务器发起两次请求，第一次是浏览器使用OPTIONS方法发起一个预检请求，
			// 第二次才是真正的异步请求，第一次的预检请求获知服务器是否允许该跨域请求：如果允许，才发起第二次真实的请求；
			// 如果不允许，则拦截第二次请求。
			httpServletResponse.setHeader("Access-Control-Allow-Methods", "GET,POST,PUT,DELETE,OPTIONS");
			//用来指定本次预检请求的有效期，单位为秒，，在此期间不用发出另一条预检请求。
			httpServletResponse.setHeader("Access-Control-Max-Age", "3600");
			//是否允许请求带有验证信息，若要获取客户端域下的cookie时，需要将其设置为true；
			httpServletResponse.setHeader("Access-Control-Allow-Credentials", "true");
			//允许服务端访问的客户端请求头，多个请求头用逗号分割，例如：Content-Type；
			httpServletResponse.setHeader("Access-Control-Allow-Headers", "Content-Type,X-Token");
			//Access-Control-Expose-Headers:该字段可选。CORS请求时，XMLHttpRequest对象的getResponseHeader()方法只能拿到6个基本字段：Cache-Control、Content-Language、
			// Content-Type、Expires、Last-Modified、Pragma。如果想拿到其他字段，就必须在Access-Control-Expose-Headers里面指定。
			log.debug("com/how2java/springboot/filter");
			filterChain.doFilter(servletRequest,httpServletResponse);

		}

		@Override
		public void init(FilterConfig filterConfig) throws ServletException {
			log.debug("我是filter");
		}

		@Override
		public void destroy() {

		}
	}
```
到此，一个简单的前后端分离项目就实现了
























