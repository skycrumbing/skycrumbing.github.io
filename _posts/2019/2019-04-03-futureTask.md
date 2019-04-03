---
layout: post
title: FutureTask
tags:
- thread
categories: java
description: FutureTask
---
通过直接继承Thread或者实现Runable的方法创建的线程都无法获得其返回值。为了解决这个问题，可以通过FutureTask。  

<!-- more -->

## 实现接口  
FutureTask实现了RunnableFuture接口，RunnableFuture实现了Runnable, Future接口。
所以FutureTask拥有Runnable和Future的特性。  
### Runnable  
是为了让FutureTask拥有线程的特性。让FutureTask像线程一样被执行。  
### Future  
Future的目的是获得和改变FutureTask执行过程中的特征。包括是否执行完成，是否需要
取消执行以及获得执行结果。  
那么在FutureTask中如何产生结果让Future调用呢？  
这就需要在创建FutureTask时传入一个**Callable**。  
## 代码
```
package com.tantao;

import java.util.concurrent.*;

/**
 * Created by Administrator on 2019/3/22.
 */
public class FutureTaskDemo {
    public static void main(String[] args) {
        FutureTask<String> futureTask = new FutureTask<String>(new Callable<String>() {
            @Override
            public String call() throws Exception {
                Thread.sleep(5000);
                return "I am result";
            }
        });
        ExecutorService executor = Executors.newSingleThreadExecutor();
        executor.submit(futureTask);
        System.out.println("得到线程结果：");
        try {
            System.out.println(futureTask.isDone());
            //会阻塞
            String s = futureTask.get();
            System.out.println(futureTask.isDone());
            System.out.println(s);
        } catch (Exception e) {
            e.printStackTrace();
        }finally {
            executor.shutdown();
        }
    }
}
```
输出：  
![结果](\assets\img\futureTask_1.jpg)  


