---
layout: post
title: synchronized关键字
tags:
- thread
categories: java
description: java线程
---
## Synchronized  
synchronized是一个Java关键字。是一种同步锁和可重入锁。可以保证线程的安全问题。  

<!-- more -->

## 修饰方法
**多个线程访问单个对象**  
```
	package com.tantao;

	/**
	 * Created by Administrator on 2019/2/20.
	 */
	public class MyThread1 extends Thread{

		public static void main(String[] args) {
			SetNum setNum = new SetNum();
			Thread1 thread1 = new Thread1(setNum);
			Thread2 thread2 = new Thread2(setNum);
			thread1.start();
			thread2.start();
		}
	}

	class Thread1 extends Thread{
		private  SetNum setNum;
		public Thread1(SetNum setNum){
			this.setNum = setNum;
		}
		@Override
		public void run(){
			try {
				setNum.setNumByString("a");
			} catch (InterruptedException e) {
				e.printStackTrace();
			}
		}
	}

	class Thread2 extends Thread{
		private  SetNum setNum;
		public Thread2(SetNum setNum){
			this.setNum = setNum;
		}
		@Override
		public void run(){
			try {
				setNum.setNumByString("adf");
			} catch (InterruptedException e) {
				e.printStackTrace();
			}
		}
	}

	class SetNum{
		private int num = 0;
		synchronized public void setNumByString(String c) throws InterruptedException {
			if("a".equals(c)){
				num = 100;
				System.out.println("a is seted");
				//如果是a让线程休眠2秒
				Thread.sleep(2000);
			}else{
				num = 200;
				System.out.println("other is seted");
			}
			System.out.println(c + " num=" + num);
		}
	}
```
结果正常，线程1先执行完，线程二再执行  
![结果](\assets\img\synchronized_1.jpg)  
**多个线程访问多个对象**  
```
	package com.tantao;

	/**
	 * Created by Administrator on 2019/2/20.
	 */
	public class MyThread1 extends Thread{

		public static void main(String[] args) {
	//        SetNum setNum = new SetNum();
			SetNum setNum1 = new SetNum();
			SetNum setNum2 = new SetNum();
			Thread1 thread1 = new Thread1(setNum1);
			Thread2 thread2 = new Thread2(setNum2);
			thread1.start();
			thread2.start();
		}
	}

	class Thread1 extends Thread{
		private  SetNum setNum;
		public Thread1(SetNum setNum){
			this.setNum = setNum;
		}
		@Override
		public void run(){
			try {
				setNum.setNumByString("a");
			} catch (InterruptedException e) {
				e.printStackTrace();
			}
		}
	}

	class Thread2 extends Thread{
		private  SetNum setNum;
		public Thread2(SetNum setNum){
			this.setNum = setNum;
		}
		@Override
		public void run(){
			try {
				setNum.setNumByString("adf");
			} catch (InterruptedException e) {
				e.printStackTrace();
			}
		}
	}

	class SetNum{
		private int num = 0;
		synchronized public void setNumByString(String c) throws InterruptedException {
			if("a".equals(c)){
				num = 100;
				System.out.println("a is seted");
				//如果是a让线程休眠2秒
				Thread.sleep(2000);
			}else{
				num = 200;
				System.out.println("other is seted");
			}
			System.out.println(c + " num=" + num);
		}
	}
```
结果显示线程1和线程2是异步执行的  
![结果](\assets\img\synchronized_2.jpg)  
**原因**  
synchronized取得的锁都是对象锁，而不是把方法当成锁。如果多个线程访问一个对象的方法，那么多个线程会去抢那个对象锁，抢到锁的线程才会执行，其他线程则会等待。如果对象是多个，就会有多个线程抢到不同的对象锁，然后并发执行。
## 可重入锁
自己可以再次获取自己的内部锁而不会造成死锁  
```
	package com.tantao;

	/**
	 * Created by Administrator on 2019/2/20.
	 */
	public class MyThread2 extends Thread{
		@Override
		public void run(){
			Server server = new Server();
			server.server1();
		}

		public static void main(String[] args) {
			MyThread2 myThread2 = new MyThread2();
			myThread2.start();
		}
	}

	class Server{
		synchronized public void server1(){
			System.out.println("server1");
			server2();
		}

		synchronized public void server2(){
			System.out.println("server2");
			server3();
		}

		synchronized public void server3(){
			System.out.println("server3");
		}
	}
```
结果  
![结果](\assets\img\synchronized_3.jpg)  
**可重入锁支持继承体系**
```
	package com.tantao;

	/**
	 * Created by Administrator on 2019/2/20.
	 */
	public class MyThread3 extends Thread{
		private Sub sub;
		@Override
		public void run(){
			sub = new Sub();
			sub.subPrint();
		}

		public static void main(String[] args) {
			MyThread3 myThread3 = new MyThread3();
			myThread3.start();
		}
	}

	 class Super {
		public int i = 10;
		synchronized public void superPrint(){
			i--;
			System.out.println("Super print i = "  + i);
		}
	 }

	 class Sub extends Super{
		 synchronized public void subPrint(){
			for(; i > 0; i--){
				System.out.println("Sub print i = "  + i);
				//休眠100毫秒
				try {
					Thread.sleep(100);
				} catch (InterruptedException e) {
					e.printStackTrace();
				}
				this.superPrint();
			}
		}
	 }
```
**结果**  
![结果](\assets\img\synchronized_4.jpg)  
**同步不具有继承性**
父类带synchronized关键字的方法被子类重写了，还是需要在方法前加synchronized关键字  
## 修饰语句块
**Synchronized修饰方法的缺点**  
在线程执行的方法中有些IO操作很耗时，也不需要同步，但是使用Synchronized修饰方法却必须整个方法同步  
```
	package com.tantao;

	/**
	 * Created by Administrator on 2019/2/20.
	 */
	public class MyThread4  {
		public static void main(String[] args) throws InterruptedException {
			Task task = new Task();
			Thread_1 thread_1 = new Thread_1(task);
			Thread_2 thread_2 = new Thread_2(task);
			thread_1.start();
			thread_2.start();
			//计算两个线程的执行时间
			Thread.sleep(10000);
			long begin = UtilGetTime.begin1 < UtilGetTime.begin2 ? UtilGetTime.begin1: UtilGetTime.begin2 ;
			long end = UtilGetTime.end1 > UtilGetTime.end2 ? UtilGetTime.end1: UtilGetTime.end2;
			System.out.println("时间：" + (end - begin));
		}
	}

	class Thread_1 extends Thread{
		private Task task;
		public Thread_1(Task task){
			this.task = task;
		}
		@Override
		public void run(){
			UtilGetTime.begin1 = System.currentTimeMillis();
			try {
				task.doTask();
			} catch (InterruptedException e) {
				e.printStackTrace();
			}
			UtilGetTime.end1 = System.currentTimeMillis();
		}
	}

	class Thread_2 extends Thread{
		private Task task;
		public Thread_2(Task task){
			this.task = task;
		}
		@Override
		public void run(){
			UtilGetTime.begin2 = System.currentTimeMillis();
			try {
				task.doTask();
			} catch (InterruptedException e) {
				e.printStackTrace();
			}
			UtilGetTime.end2 = System.currentTimeMillis();
		}
	}

	/*
	 *线程执行的具体代码
	 */
	class Task{
		private String getData1;
		private String getData2;

		synchronized public void doTask() throws InterruptedException {
			System.out.println("begin task:");
			//不需要同步的部分
			Thread.sleep(3000);
			String privateGetData1  = "长时间处理任务后从远程返回的值1 threadName="
					+ Thread.currentThread().getName();
			String privateGetData2  = "长时间处理任务后从远程返回的值2 threadName="
					+ Thread.currentThread().getName();

			//需要同步的部分
			getData1 = privateGetData1;
			getData2 = privateGetData2;

			//不需要同步的部分
			System.out.println(getData1);
			System.out.println(getData2);
			System.out.println("end task");
		}
	}
	/*
	 *用于获取两个线程的开始时间和结束时间
	 */
	class UtilGetTime{
		public static long begin1;
		public static long begin2;

		public static long end1;
		public static long end2;
	}
```
**结果**  
![结果](\assets\img\synchronized_5.jpg)  
### Synchronized(this)修饰需要同步的代码块
1，Synchronized(this)代码块持有当前对象锁。不在synchronized代码块中就异步执行，在synchronized代码块中就是同步执行  
2，当一个对象访问synchronized(this)代码块时，其他线程对同一个对象中所有其他synchronized(this)代码块代码块的访问将被阻塞  
3，synchronized关键字加到static静态方法和synchronized(class)加到代码块上都是是给Class类上锁  
改进doTask方法  
```
     public void doTask() throws InterruptedException {
        System.out.println("begin task:");
        //不需要同步的部分
        Thread.sleep(3000);
        String privateGetData1  = "长时间处理任务后从远程返回的值1 threadName="
                + Thread.currentThread().getName();
        String privateGetData2  = "长时间处理任务后从远程返回的值2 threadName="
                + Thread.currentThread().getName();

        //需要同步的部分
         synchronized (this){
             getData1 = privateGetData1;
             getData2 = privateGetData2;
         }

        //不需要同步的部分
        System.out.println(getData1);
        System.out.println(getData2);
        System.out.println("end task");
    }
```
**结果**  
![结果](\assets\img\synchronized_6.jpg)  
注意：因为数据类型String的常量池属性，所以synchronized(string)在使用时某些情况下会出现一些问题，比如两个线程运行 synchronized(“abc”)｛ ｝和 synchronized(“abc”)｛ ｝修饰的方法时，这两个线程就会持有相同的锁，导致某一时刻只有一个线程能运行。所以尽量不要使用synchronized(string)而使用synchronized(object)


