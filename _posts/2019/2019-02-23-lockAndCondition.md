---
layout: post
title: Lock/Condition
tags:
- thread
categories: java
description: java线程
---
## Lock接口  
在Lock接口出现之前，Java程序是靠synchronized关键字实现锁功能的。JDK1.5之后并发包中新增了Lock接口以及相关实现类来实现锁功能，使得线程之间的处理能更灵活。  

<!-- more -->

## Lock接口的实现
ReentrantLock ， ReentrantReadWriteLock.ReadLock ， ReentrantReadWriteLock.WriteLock  
ReentrantLock：重入锁，同时是一种排它锁，同一时刻只允许一个线程访问  
### Lock用法
```
	Lock lock=new ReentrantLock()；
	  lock.lock();
	   try{
		  doSomething()
		}finally{
		lock.unlock();
		}
```
finall语句块中释放锁的目的是保证获取到锁之后，最终能够被释放。  
### 实例   
```
	package com.tantao;

	import java.util.concurrent.locks.Lock;
	import java.util.concurrent.locks.ReentrantLock;

	/**
	 * Created by Administrator on 2019/2/21.
	 */
	public class ThreadLock {
		public static void main(String[] args) {
			Service service = new Service();
			MyThread threadA = new MyThread("A", service);
			MyThread threadB = new MyThread("B", service);
			MyThread threadC = new MyThread("C", service);
			threadA.start();
			threadB.start();
			threadC.start();
		}
	}

	class MyThread extends Thread{
		private Service service;
		public MyThread(String name, Service service){
			this.setName(name);
			this.service = service;
		}
		@Override
		public void run(){
			service.doService();
		}
	}

	class Service{
		private Lock lock = new ReentrantLock();
		public void doService(){
			lock.lock();
			try{
				for(int i = 0; i < 5; i ++){
					System.out.println("ThreadName:" + Thread.currentThread().getName() + "----" + "i = " + i );
				}
			}finally {
				lock.unlock();
			}
		}

	}
```
结果  
![结果](\assets\img\lockAndCondition_1.jpg)  
## Condition接口  
synchronized关键字与wait()和notify/notifyAll()方法相结合可以实现等待/通知机制。  
同样Lock和Condition也能够实现，并且灵活性也更高。  
比如可以实现多路通知功能。也就是在一个Lock对象中可以创建多个Condition实例（即对象监视器），线程对象可以注册在指定的Condition中，从而可以有选择性的进行线程通知，在调度线程上更加灵活。  
使用ReentrantLock类结合Condition实例可以实现“选择性通知”  
### 单个Condition实例  
```
	package com.tantao;

	import java.util.concurrent.locks.Condition;
	import java.util.concurrent.locks.Lock;
	import java.util.concurrent.locks.ReentrantLock;

	/**
	 * Created by Administrator on 2019/2/21.
	 */
	public class ConditionDemo {
		public static void main(String[] args) throws InterruptedException {
			MyService myService = new MyService();
			TestThread thread = new TestThread(myService);
			thread.start();
			Thread.sleep(3000);
			myService.signal();
		}
	}

	class MyService{
		private Lock lock = new ReentrantLock();
		public Condition condition = lock.newCondition();

		public void await(){
			lock.lock();
			try{
				System.out.println("开始await，进入等待队列，时间：" + System.currentTimeMillis());
				condition.await();
				System.out.println("被signal通知，继续执行，时间" + System.currentTimeMillis());
			} catch (InterruptedException e) {
				e.printStackTrace();
			}finally {
				lock.unlock();
			}
		}

		public void signal() throws InterruptedException {
			lock.lock();
			try {
				System.out.println("signal时间为" + System.currentTimeMillis());
				condition.signal();
				Thread.sleep(3000);
				System.out.println("这是condition.signal()方法之后的语句,要把代码块中的内容执行完才能继续执行await中的方法");
			} finally {
				lock.unlock();
			}
		}
	}

	class TestThread extends Thread{
		private MyService service;

		public TestThread(MyService service) {
			super();
			this.service = service;
		}

		@Override
		public void run() {
			service.await();
		}
	}
```
结果  
![结果](\assets\img\lockAndCondition_2.jpg)  
### 多个Condition实例  
```
	package com.tantao;

	import java.util.concurrent.locks.Condition;
	import java.util.concurrent.locks.Lock;
	import java.util.concurrent.locks.ReentrantLock;

	/**
	 * Created by Administrator on 2019/2/21.
	 */
	public class ConditionDemo_2 {
		public static void main(String[] args) throws InterruptedException {
			MyService_2 service = new MyService_2();
			Thread_A threadA = new Thread_A(service, "A");
			Thread_B threadB = new Thread_B(service, "B");

			threadA.start();
			threadB.start();

			Thread.sleep(3000);
			service.signalAll_A();
		}
	}

	class Thread_A extends Thread {
		private MyService_2 service;

		public Thread_A(MyService_2 service,String name) {
			this.service = service;
			this.setName(name);
		}

		@Override
		public void run(){
			service.awaitA();
		}
	}

	class Thread_B extends Thread {
		private MyService_2 service;

		public Thread_B(MyService_2 service,String name) {
			this.service = service;
			this.setName(name);
		}

		@Override
		public void run(){
			service.awaitB();
		}
	}

	class MyService_2 {
		private Lock lock = new ReentrantLock();
		public Condition conditionA = lock.newCondition();
		public Condition conditionB = lock.newCondition();

		public void awaitA() {
			lock.lock();
			try {
				System.out.println("begin awaitA时间为" + System.currentTimeMillis()
						+ " ThreadName=" + Thread.currentThread().getName());
				conditionA.await();
				System.out.println("end awaitA时间为" + System.currentTimeMillis() + " ThreadName=" + Thread.currentThread().getName());
			} catch (InterruptedException e) {
				e.printStackTrace();
			} finally {
				lock.unlock();
			}
		}

		public void awaitB() {
			lock.lock();
			try {
				System.out.println("begin awaitB时间为" + System.currentTimeMillis()
						+ " ThreadName=" + Thread.currentThread().getName());
				conditionB.await();
				System.out.println("  end awaitB时间为" + System.currentTimeMillis()
						+ " ThreadName=" + Thread.currentThread().getName());
			} catch (InterruptedException e) {
				e.printStackTrace();
			} finally {
				lock.unlock();
			}
		}

		public void signalAll_A() {
			lock.lock();
			try {
				System.out.println("  signalAll_A时间为" + System.currentTimeMillis()
						+ " ThreadName=" + Thread.currentThread().getName());
				conditionA.signalAll();
			} finally {
				lock.unlock();
			}
		}

		public void signalAll_B() {
			lock.lock();
			try {
				System.out.println("  signalAll_B时间为" + System.currentTimeMillis()
						+ " ThreadName=" + Thread.currentThread().getName());
				conditionB.signalAll();
			} finally {
				lock.unlock();
			}
		}
	}
```
结果  
![结果](\assets\img\lockAndCondition_3.jpg)  
B线程由于没有唤醒，一直在等待队列中，程序无法停止  
### Condition实现顺序执行  
```
	package com.tantao;

	import java.util.concurrent.locks.Condition;
	import java.util.concurrent.locks.Lock;
	import java.util.concurrent.locks.ReentrantLock;

	/**
	 * Created by Administrator on 2019/2/21.
	 */
	public class ConditionDemo_3 {
		volatile private static String flag = "A";
		private static ReentrantLock lock = new ReentrantLock();
		final private static Condition conditionA = lock.newCondition();
		final private static Condition conditionB = lock.newCondition();
		final private static Condition conditionC = lock.newCondition();

		public static void main(String[] args) {
			Thread threadA = new Thread(
					()->{
						lock.lock();
						try{
							while(flag != "A"){
								//不到A运行
								conditionA.await();
							}
							for(int i = 0; i < 3; i ++){
								System.out.println("ThreadA: " + i);
							}
							flag = "B";
							//通知B运行
							conditionB.signalAll();
						} catch (InterruptedException e) {
							e.printStackTrace();
						} finally {
							lock.unlock();
						}
					}
			);

			Thread threadB = new Thread(
					()->{
						lock.lock();
						try{
							while(flag != "B"){
								conditionB.await();
							}
							for(int i = 0; i < 3; i ++){
								System.out.println("ThreadB: " + i);
							}
							flag = "C";
							conditionC.signalAll();
						} catch (InterruptedException e) {
							e.printStackTrace();
						} finally {
							lock.unlock();
						}
					}
			);

			Thread threadC = new Thread(
					()->{
						lock.lock();
						try{
							while(flag != "C"){
								conditionC.await();
							}
							for(int i = 0; i < 3; i ++){
								System.out.println("ThreadC: " + i);
							}
							flag = "A";
							conditionA.signalAll();
						} catch (InterruptedException e) {
							e.printStackTrace();
						} finally {
							lock.unlock();
						}
					}
			);



			Thread[] threadsA = new Thread[5];
			Thread[] threadsB = new Thread[5];
			Thread[] threadsC = new Thread[5];
			for(int i = 0; i < 5; i++){
				threadsA[i] = new Thread(threadA);
				threadsB[i] = new Thread(threadB);
				threadsC[i] = new Thread(threadC);
				threadsA[i].start();
				threadsB[i].start();
				threadsC[i].start();
			}

		}

	}
```
结果  
![结果](\assets\img\lockAndCondition_4.jpg)  
## 公平锁与非公平锁  
锁分为公平锁和非公平锁，公平锁是指线程获取锁的顺序是按照线程加锁的顺序分配的，即先进先出的顺序。而非公平锁就是一种获取锁的抢占机制，是随机获取锁的这样可能照成某学线程一直拿不到锁，也就是不公平了。  
synchronized只能是非公平锁，ReenTrantLock默认情况是非公平的  
```
	package com.tantao;

	import java.util.concurrent.locks.Lock;
	import java.util.concurrent.locks.ReentrantLock;

	import static javafx.scene.input.KeyCode.T;

	/**
	 * Created by Administrator on 2019/2/21.
	 */
	public class NoFairLock {

		public static void main(String[] args) {
			NoFairService service = new NoFairService();
			Thread thread = new Thread(
					() ->{
						System.out.println("★线程" + Thread.currentThread().getName()
								+ "运行了");
						service.doService();
					}
			);

			Thread[] threads = new Thread[10];
			for(int i = 0; i < 10; i++){
				threads[i] = new Thread(thread);
			}
			for (int i = 0; i < 10; i++) {
				threads[i].start();
			}
		}

	}
	class NoFairService{
		//不公平锁
		private Lock lock = new ReentrantLock(false);

		public void doService(){
			lock.lock();
			try{
				System.out.println("Thread Name: " + Thread.currentThread().getName() + "获得锁");
			}finally {
				lock.unlock();
			}
		}
	}
```
结果  
![结果](\assets\img\lockAndCondition_5.jpg)  
本来线程1先运行，结果却是线程2先获得锁，这就是不公平锁。  
## ReadWriteLock接口
ReentrantLock具有完全互斥排他的效果，即同一时刻只允许一个线程访问  
ReadWriteLock提供两种锁，一个是读操作相关的锁也成为共享锁，一个是写操作相关的锁 也称为排他锁。  
多个读锁之间不互斥，读锁与写锁互斥，写锁与写锁互斥（只要是和写相关的操作都互斥）  
### 读操作  
```
	package com.tantao;

	import java.util.concurrent.locks.Lock;
	import java.util.concurrent.locks.ReentrantLock;
	import java.util.concurrent.locks.ReentrantReadWriteLock;

	/**
	 * Created by Administrator on 2019/2/21.
	 */
	public class ReadLock {
		public static void main(String[] args) {
			ReadService service = new ReadService();
			Thread thread1 = new Thread(
					() ->{
						System.out.println("★线程" + Thread.currentThread().getName()
								+ "运行了");
						service.readService();
	//                    service.wirteService();
					}
			);
			Thread thread2 = new Thread(
					() ->{
						System.out.println("★线程" + Thread.currentThread().getName()
								+ "运行了");
						service.readService();
	//                    service.wirteService();
					}
			);
			//读操作
			thread1.start();
			thread2.start();
		}
	}

	class ReadService{
		//读锁
		private ReentrantReadWriteLock lock=  new ReentrantReadWriteLock();

		public void wirteService(){
			lock.writeLock().lock();
			try{
				System.out.println("Thread Name: " + Thread.currentThread().getName() + "获得锁开始写操作");
				Thread.sleep(5000);
				System.out.println("写操作完成");
			} catch (InterruptedException e) {
				e.printStackTrace();
			} finally {
				lock.writeLock().unlock();
			}
		}

		public void readService(){
			lock.readLock().lock();
			try{
				System.out.println("Thread Name: " + Thread.currentThread().getName() + "获得锁开始读操作");
				Thread.sleep(5000);
				System.out.println("读操作完成");
			} catch (InterruptedException e) {
				e.printStackTrace();
			} finally {
				lock.readLock().unlock();
			}
		}
	}
```
结果  
![结果](\assets\img\lockAndCondition_6.jpg)  
写操作类似就不再演示  
