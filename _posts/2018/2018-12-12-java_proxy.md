---
layout: post
title: java代理
tags:
- designPattern
categories: java
description: java代理的实现
---
## java代理
通俗的来说，一个对象A要实现一个方法，在没代理前是由对象A做，代理之后由A的代理对象B来做

<!-- more -->

## 静态代理
在程序运行前就已经存在了代理类的字节码文件，代理类和原始类的关系在运行前就已经确定。
```
	package test.staticProxy;  
	// 接口  
	public interface IUserDao {  
	 void save();  
	 void find();  
	}  
	//目标对象  
	class UserDao implements IUserDao{  
	 @Override  
	 public void save() {  
	   System.out.println("模拟：保存用户！");  
	 }  
	 @Override  
	 public void find() {  
	   System.out.println("模拟：查询用户");  
	 }  
	}  
	/** 
	   静态代理 
			 特点： 
	 1. 目标对象必须要实现接口 
	 2. 代理对象，要实现与目标对象一样的接口 
	*/  
	class UserDaoProxy implements IUserDao{  
	 // 代理对象，需要维护一个目标对象  
	 private IUserDao target = new UserDao();  
	 @Override  
	 public void save() {  
	   System.out.println("代理操作： 开启事务...");  
	   target.save();   // 执行目标对象的方法  
	   System.out.println("代理操作：提交事务...");  
	 }  
	 @Override  
	 public void find() {  
	   target.find();  
	 }  
	}  
```

静态代理虽然保证了业务类只需关注逻辑本身，**代理对象的一个接口只服务于一种类型的对象，如果要代理的方法很多，势必要为每一种方法都进行代理。再者，如果增加一个方法，除了实现类需要实现这个方法外，所有的代理类也要实现此方法。增加了代码的维护成本**。那么要如何解决呢?答案是使用动态代理。
## 动态代理模式
动态代理类的源码是在程序运行期间通过JVM反射等机制动态生成，代理类和委托类的关系是运行时才确定的。  
```
	package test.staticProxy;  
	package test.dynamicProxy;

	import java.lang.reflect.InvocationHandler;
	import java.lang.reflect.Method;
	import java.lang.reflect.Proxy;
	// 接口
	public interface IUserDao {
	 void save();
	 void find();
	}
	//目标对象
	class UserDao implements IUserDao{
	 @Override
	 public void save() {
	   System.out.println("模拟： 保存用户！");
	 }
	 @Override
	 public void find() {
	   System.out.println("查询");
	 }
	}
	/**
	* 动态代理：
	*    代理工厂，给多个目标对象生成代理对象！
	*
	*/
	class ProxyFactory {
	 // 接收一个目标对象
	 private Object target;
	 public ProxyFactory(Object target) {
	   this.target = target;
	 }
	 // 返回对目标对象(target)代理后的对象(proxy)
	 public Object getProxyInstance() {
	   Object proxy = Proxy.newProxyInstance(
		 target.getClass().getClassLoader(),  // 目标对象使用的类加载器
		 target.getClass().getInterfaces(),   // 目标对象实现的所有接口
		 new InvocationHandler() {      // 执行代理对象方法时候触发
		   @Override
		   public Object invoke(Object proxy, Method method, Object[] args)
			   throws Throwable {

			 // 获取当前执行的方法的方法名
			 String methodName = method.getName();
			 // 方法返回值
			 Object result = ;
			 if ("find".equals(methodName)) {
			   // 直接调用目标对象方法
			   result = method.invoke(target, args);
			 } else {
			   System.out.println("开启事务...");
			   // 执行目标对象方法
			   result = method.invoke(target, args);
			   System.out.println("提交事务...");
			 }
			 return result;
		   }
		 }
	   );
	   return proxy;
	 }
	}
```
测试方法如下：  
```
	public static void main(String[], args){
		//目标对象
		IuserDao target = new UserDao();
		System.out.println("目标对象" + target.getClass());
		//代理对象
		IuserDao proxy = (IuserDao) new proxFactory(target).getProxyInstance();
		system.out.println("代理对象" + proxy.getClass());
		//执行代理对象的方法
		proxy.save()
	}
```
**使用jdk生成的动态代理的前提是目标类必须有实现的接口**。但这里又引入一个问题,如果某个类没有实现接口,就不能使用JDK动态代理,所以Cglib代理就是解决这个问题的。
## Cglib代理
Cglib是以动态生成的子类继承目标的方式实现，在运行期动态的在内存中构建一个子类。  
Cglib使用的前提是目标类不能为final修饰。因为final修饰的类不能被继承。  
## spring AOP
通过定义和前面代码我们可以发现3点：  
1.AOP是基于动态代理模式。  
2.AOP是方法级别的（要测试的方法不能为static修饰，因为接口中不能存在静态方法，编译就会报错）。  
3.AOP可以分离业务代码和关注点代码（重复代码），在执行业务代码时，动态的注入关注点代码。切面就是关注点代码形成的类。  
**存在接口则用动态代理，无接口则使用Cgilb代理。如果目标类没有实现接口，且class为final修饰的，则不能进行Spring AOP编程！**





