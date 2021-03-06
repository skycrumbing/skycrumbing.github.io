---
layout: post
title: 网络编程IOvsNIO
tags:
- io
categories: network
description: 网络编程IOvsNIO
---
## 网络编程
网络编程有IO与NIO两种方式,NIO与传统IO有什么区别，我将会通过编程的形似展示出来

<!-- more -->

## 传统IO编程
服务端代码
```
	public class IoServer {
		public static void main(String[] args) throws Exception{
			ServerSocket serverSocket = new ServerSocket(8000);

			// (1) 接收新连接线程
			new Thread(() -> {
				while (true) {
					try {
						// (1) 阻塞方法获取新的连接
						Socket socket = serverSocket.accept();
						// (2) 每一个新的连接都创建一个线程，负责读取数据
						new Thread(() -> {
							try {
								byte[] data = new byte[1024];
								InputStream inputStream = socket.getInputStream();
								while (true) {
									int len;
									// (3) 按字节流方式读取数据
									while ((len = inputStream.read(data)) != -1) {
										System.out.println(new String(data, 0, len));
									}
								}
							} catch (IOException e) {
							}
						}).start();
					} catch (IOException e) {
					}
				}
			}).start();
		}
	}
```
客户端代码  
```
	public class IOClient {
		public static void main(String[] args) {
			new Thread(() -> {
				try {
					Socket socket = new Socket("127.0.0.1", 8000);
					while (true) {
						try {
							socket.getOutputStream().write((new Date() + ": hello world").getBytes());
							socket.getOutputStream().flush();
							Thread.sleep(2000);
						} catch (Exception e) {
						}
					}
				} catch (IOException e) {
				}
			}).start();
		}
	}
```
先启动服务端，再启动客户端，服务端控制台显示结果：  
![结果](\assets\img\networkCode_1.jpg)
上面的demo，从服务端代码中我们可以看到，在传统的IO模型中，每个连接创建成功之后都需要一个线程来维护，每个线程包含一个while死循环读取数据，那么1w个连接对应1w个线程，继而1w个while死循环，这就带来如下几个问题：  
1，线程资源受限：线程是操作系统中非常宝贵的资源，同一时刻有大量的线程处于阻塞状态是非常严重的资源浪费，操作系统耗不起  
2，线程切换效率低下：单机cpu核数固定，线程爆炸之后操作系统频繁进行线程切换，应用性能急剧下降。  
3，除了以上两个问题，IO编程中，我们看到数据读写是以字节流为单位，效率不高。  
为了解决这三个问题，JDK在1.4之后提出了NIO。  
## NIO与IO对比
![NIO与IO对比图](\assets\img\networkCode_2.jpg)
- 传统IO模型中，一个连接来了，会创建一个线程，对应一个while死循环，死循环的目的就是不断监测这条连接上是否有数据可以读，大多数情况下，1w个连接里面同一时刻只有少量的连接有数据可读，因此，很多个while死循环都白白浪费掉了，因为读不出啥数据。  
- 在NIO模型中，他把这么多while死循环变成一个死循环，这个死循环由一个线程控制，那么他又是如何做到一个线程，一个while死循环就能监测1w个连接是否有数据可读的呢。  
- 这就是NIO模型中selector的作用，一条连接来了之后，现在不创建一个while死循环去监听是否有数据可读了，而是直接把这条连接注册到selector上，然后，通过检查这个selector，就可以批量监测出有数据可读的连接，进而读取数据。  
## NIO编程
服务端代码  
```
	public class NIOServer {
		public static void main(String[] args) throws IOException {
			Selector serverSelector = Selector.open();
			Selector clientSelector = Selector.open();

			new Thread(() -> {
				try {
					// 对应IO编程中服务端启动
					ServerSocketChannel listenerChannel = ServerSocketChannel.open();
					listenerChannel.socket().bind(new InetSocketAddress(8000));
					listenerChannel.configureBlocking(false);
					listenerChannel.register(serverSelector, SelectionKey.OP_ACCEPT);

					while (true) {
						// 监测是否有新的连接，这里的1指的是阻塞的时间为1ms
						if (serverSelector.select(1) > 0) {
							Set<SelectionKey> set = serverSelector.selectedKeys();
							Iterator<SelectionKey> keyIterator = set.iterator();

							while (keyIterator.hasNext()) {
								SelectionKey key = keyIterator.next();

								if (key.isAcceptable()) {
									try {
										// (1) 每来一个新连接，不需要创建一个线程，而是直接注册到clientSelector
										SocketChannel clientChannel = ((ServerSocketChannel) key.channel()).accept();
										clientChannel.configureBlocking(false);
										clientChannel.register(clientSelector, SelectionKey.OP_READ);
									} finally {
										keyIterator.remove();
									}
								}

							}
						}
					}
				} catch (IOException ignored) {
				}

			}).start();


			new Thread(() -> {
				try {
					while (true) {
						// (2) 批量轮询是否有哪些连接有数据可读，这里的1指的是阻塞的时间为1ms
						if (clientSelector.select(1) > 0) {
							Set<SelectionKey> set = clientSelector.selectedKeys();
							Iterator<SelectionKey> keyIterator = set.iterator();

							while (keyIterator.hasNext()) {
								SelectionKey key = keyIterator.next();

								if (key.isReadable()) {
									try {
										SocketChannel clientChannel = (SocketChannel) key.channel();
										ByteBuffer byteBuffer = ByteBuffer.allocate(1024);
										// (3) 读取数据以块为单位批量读取
										clientChannel.read(byteBuffer);
										byteBuffer.flip();
										System.out.println(Charset.defaultCharset().newDecoder().decode(byteBuffer)
												.toString());
									} finally {
										keyIterator.remove();
										key.interestOps(SelectionKey.OP_READ);
									}
								}

							}
						}
					}
				} catch (IOException ignored) {
				}
			}).start();


		}
```
客户端代码不变  
显示结果也不变  
### 代码详解
NIO模型中通常会有两个线程，每个线程绑定一个轮询器selector，在这个例子中serverSelector负责轮询是否有新的连接，clientSelector负责轮询连接是否有数据可读 服务端监测到新的连接之后，不再创建一个新的线程，而是直接将新连接绑定clientSelector上，这样就不用IO模型中1w个while循环在死等，参见(1) clientSelector被一个while死循环包裹着，如果在某一时刻有多条连接有数据可读，那么通过 clientSelector.select(1)方法可以轮询出来，进而批量处理，参见(2)数据的读写以内存块为单位，参见(3)
### 优势
- 线程资源：实际开发过程中，会开多个线程，每个线程都管理着一批连接，相对于IO模型中一个线程管理一条连接，消耗的线程资源大幅减少  
- 线程切换效率：由于NIO模型中线程数量大大降低，线程切换效率因此也大幅度提高。  
- IO读写以字节为单位：NIO解决这个问题的方式是数据读写不再以字节为单位，而是以字节块为单位。IO模型中，每次都是从操作系统底层一个字节一个字节地读取数据，而NIO维护一个缓冲区，每次可以从这个缓冲区里面读取一块的数据
## Netty编程
Netty封装了JDK的NIO，让代码更简洁，并且增加了很多NIO编程的细节处理，让开发变得更加容易。用官方正式的话来说就是：Netty是一个异步事件驱动的网络应用框架，用于快速开发可维护的高性能服务器和客户端。  





