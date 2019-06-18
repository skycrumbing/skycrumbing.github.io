---
layout: post
title: java中retry的使用
tags:
- keyword
categories: java
description: java中retry的使用
---
## retry
retry就是一个标记，也可以使用其他名称命名，如retry1，retry2。标记程序跳出循环的时候从哪里开始执行（功能类似于goto，不懂）。

<!-- more -->

## 使用
### 场景一
```
public class TestRetry {
    public static void main(String[] args) {
        retry:
        for(int i = 0; i < 10; i++){
            while(i == 5){
                continue retry;
            }
            System.out.println(i);
        }
    }
}
```
结果：输出 0 1 2 3 4 6 7 8 9  
原因：当i==5时，程序跳到第4行，此时i==5，执行i++。  
### 场景二  
```
public class TestRetry {
    public static void main(String[] args) {
        for(int i = 0; i < 10; i++){
            retry:
            while(i == 5){
                continue retry;
            }
            System.out.println(i);
        }
    }
}
```
结果：输出 0 1 2 3 4  
原因：当i==5时，程序跳到第5行，此时i==5，又跳到第5行，进入死循环。  
## 总结
retry一般都是跟随者for循环出现，第一个retry的下面一行就是for循环，而且第二个retry的前面一般是 continue或是 break。

