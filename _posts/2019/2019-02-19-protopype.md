---
layout: post
title: 原型设计模式
tags:
- designPattern
categories: java
description:  原型设计模式
---
## 原型模式
原型模式是用来克隆自身对象，从而克隆出多个与原型对象一摸一样的对象。  
在Spring中，用户也可以采用原型模式来创建新的bean实例，从而实现每次获取的是通过克隆生成的新实例，对其进行修改时对原有实例对象不造成任何影响。  

<!-- more -->


在原型模式中有三类角色  
Client：使用client实现原型类的复制  
Prototype：抽象原型类，负责定义原型类的自我复制方法  
ConcretePrototype：原型类，负责实现自我复制的方法  
流程：  
原型实例将自己注册到Client中，在使用时Client通过调用原型实例的自我复制方法复制出克隆对象。  
## 克隆
### 浅克隆  
复制出的对象所有变量都与原对象的变量相同，也就是说对其他对象的引用任然指向原来的对象，而不是重新复制原对象所引用的对象。  
![浅克隆](\assets\img\protopype_1.jpg)  
### 深克隆  
复制出的对象除了引用变量其他变量都与原对象的变量相同，引用对象将会被重新复制并且指向复制对象  
![深克隆](\assets\img\protopype_2.jpg)  
### clone()
要想实现克隆，就需要实现Cloneable接口，以及实现Cloneable接口的clone方法  
### 浅克隆代码  
Person  
```
	package com.tanao.pojo;

	/**
	 * Created by Administrator on 2019/2/18.
	 */
	public class Person implements Cloneable {
		public String name;
		public Address address;

		public Person(String name, Address adress) {
			this.name = name;
			this.address = adress;
		}

		public String getName() {
			return name;
		}

		public void setName(String name) {
			this.name = name;
		}

		public Address getAddress() {
			return address;
		}

		public void setAddress(Address adress) {
			this.address = address;
		}

		@Override
		public Object clone() throws CloneNotSupportedException {
			return super.clone();
		}

	}
```
Address  
```
	package com.tanao.pojo;

	/**
	 * Created by Administrator on 2019/2/18.
	 */
	public class Address {
		public String city;
		public Address() {

		}
		public Address(String city) {
			this.city = city;
		}

		public void setCity(String city) {
			this.city = city;
		}

		public String getCity() {
			return city;
		}

	}
```
TestCopy  
```
	package com.tanao.pojo;

	import com.tanao.pojo.Address;
	import com.tanao.pojo.Person;

	/**
	 * Created by Administrator on 2019/2/18.
	 */
	public class TestCopy {
		public static void main(String[] args) {
			//new one person
			Address adress = new Address();
			adress.setCity("beijin");
			Person p1 = new Person("xiaoming", adress);

			Person p2 = null;
			try {
				p2 = (Person) p1.clone();
			} catch (CloneNotSupportedException e) {
				e.printStackTrace();
			}
			System.out.println(p1 == p2);
			System.out.println(p1.getAddress() == p2.getAddress());
		}
	}
```
结果  
![结果](\assets\img\protopype_3.jpg)  
#### 深克隆
Person  
```
	package com.tanao.pojo;

	/**
	 * Created by Administrator on 2019/2/18.
	 */
	public class Person implements Cloneable {
		public String name;
		public Address address;

		public Person(String name, Address adress) {
			this.name = name;
			this.address = adress;
		}

		public String getName() {
			return name;
		}

		public void setName(String name) {
			this.name = name;
		}

		public Address getAddress() {
			return address;
		}

		public void setAddress(Address adress) {
			this.address = address;
		}

		@Override
		public Object clone() throws CloneNotSupportedException {
			Object obj=super.clone();
			Address a=((Person)obj).getAddress();
			//这边不能用((Person)obj).setAddress((Address) a.clone())
			((Person)obj).address = (Address) a.clone();
			return obj
		}

	}
```
Address  
```
	package com.tanao.pojo;

	/**
	 * Created by Administrator on 2019/2/18.
	 */
	public class Address implements Cloneable{
		public String city;
		public Address() {

		}
		public Address(String city) {
			this.city = city;
		}

		public void setCity(String city) {
			this.city = city;
		}

		public String getCity() {
			return city;
		}

		@Override
		public Object clone() throws CloneNotSupportedException {
		   return super.clone();
		}

	}
```
TestCopy不变  
```
	package com.tanao.pojo;

	import com.tanao.pojo.Address;
	import com.tanao.pojo.Person;

	/**
	 * Created by Administrator on 2019/2/18.
	 */
	public class TestCopy {
		public static void main(String[] args) {
			//new one person
			Address adress = new Address();
			adress.setCity("beijin");
			Person p1 = new Person("xiaoming", adress);

			Person p2 = null;
			try {
				p2 = (Person) p1.clone();
			} catch (CloneNotSupportedException e) {
				e.printStackTrace();
			}
			System.out.println(p1 == p2);
			System.out.println(p1.getAddress() == p2.getAddress());
		}
	}
```
结果  
![结果](\assets\img\protopype_4.jpg)  
## 原型设计模式代码实现  
Prototype（这里用的product）  
```
	/**
	 * Created by Administrator on 2019/2/18.
	 */
	//`Cloneable接口只是起到告诉程序可以调用clone方法的作用，clone()可以实现深拷贝，它本身并没有定义任何方法
	public interface Product extends Cloneable {
		void use(String s);
		//克隆方法
		Product createClone();
	}
```
ConcretePrototype  
Box1
```
	/**
	 * Created by Administrator on 2019/2/18.
	 */
	public class Box1 implements Product {
		private char decochar;
		public Box1(char decochar){
			this.decochar = decochar;
		}
		//字符串上下左右被decochar包围
		@Override
		public void use(String s) {
			int length = s.getBytes().length;
			for(int i = 0; i < length + 4; i++){
				System.out.print(decochar);
			}
			System.out.println();

			System.out.print(decochar + "" + s + "" + decochar);
			System.out.println();

			for(int i = 0; i < length + 4; i++){
				System.out.print(decochar);
			}
			System.out.println();

		}

		//自我复制
		@Override
		public Product createClone() {
			Product p = null;
			try{
				p = (Product) clone();
			} catch (CloneNotSupportedException e) {
				e.printStackTrace();
			}
			return p;
		}
	}
```
Box2  
```
	/**
	 * Created by Administrator on 2019/2/18.
	 */
	public class Box2 implements Product{
		private char ulchar;

		public Box2(char ulchar) {
			this.ulchar = ulchar;
		}

		//字符串左右下被ulchar包围
		@Override
		public void use(String s) {
			int length = s.getBytes().length;
			System.out.println(ulchar + s + ulchar);
			for (int i = 0; i < length + 2; i++) {
				System.out.print(ulchar);
			}
			System.out.println();
		}

		@Override
		public Product createClone() {
			Product p=null;
			try {
				p=(Product) clone();
			} catch (CloneNotSupportedException e) {
				e.printStackTrace();
			}
			return p;
		}
	}
```
Client  
```
	import java.util.HashMap;

	/**
	 * Created by Administrator on 2019/2/18.
	 */
	//客户端 Manager类使用Product接口来复制实例
	public class Manager {
	   private HashMap<String, Product> showcase = new HashMap<>();
	   //将实现Product的实例注册到showcase中
	   public void register(String name, Product product){
		   showcase.put(name, product);
	   }
	   public Product create(String productNmae){
		   Product p = showcase.get(productNmae);
		   return p.createClone();
	   }
	}
```
Test  
```
	import java.net.MalformedURLException;

	/**
	 * Created by Administrator on 2019/2/18.
	 */
	public class Test {
		public static void main(String[] args) {
			Manager manager = new Manager();
			Box1 box1_1 = new Box1('~');
			Box1 b = box1_1;
			Box1 box1_2 = new Box1('*');
			Box2 box2_1 = new Box2('&');

			manager.register("test1",box1_1);
			manager.register("test2",box1_2);
			manager.register("test3",box2_1);

			Product test1 = manager.create("test1");
			test1.use("test1");
			//因为是深拷贝，所以两个对象不相等
			System.out.println(test1 == box1_1);
			System.out.println();

			Product test2 = manager.create("test2");
			test2.use("test2");
			System.out.println();

			Product test3 = manager.create("test3");
			test3.use("test3");


		}
	}
```
结果  
![结果](\assets\img\protopype_5.jpg)  
