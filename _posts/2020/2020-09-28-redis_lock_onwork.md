---
layout: post
title: 关于redis锁的使用问题  
tags:
- redis  
- lock
- AOP
categories: work
description: 在项目中使用redis锁失效  
---
## 场景  
这是一个类似淘宝抢券的业务场景，数据库中存放着当日折扣卷的数量，用户抢券逻辑如下：  
先获取当日折扣卷剩余数量；    
如果还有剩余则正常处理，发券，折扣卷数量减一；  
没有剩余则返回错误  

<!-- more -->

## 代码设计  
为了避免领取量超过预发量，使用了redis来对整个操作进行加锁  
加锁原理见之前的笔记：  
redis锁详情：[分布式锁](http://blog.tantao.site/database/2018/11/15/disLock/ "分布式锁")  

## 实际效果
当并发量大时依旧会出现领卷数量超过当日预发数量   

## 排查过程
* 首先怀疑是redis锁的问题  
将代码redis锁部分删除，在对应方法上加上synchronized关键字，用测试代码进行测试    
结果：问题依旧会出现  
* 由于加上synchronized关键字依旧无法解决问题，这时候就怀疑与spring的aop有关系了，这时候发现该方法上加上了事务的注解，于是将注解注释掉再进行了测试  
结果：使用synchronized没有问题，使用redis锁也没问题  


## 原因
@Transactional 注解使spring AOP生成了一个代理对象来对该方法做了增强，在方法执行前开始事务，在方法执行后提交事务。  
* 使用synchronized失效的原因：是因为Synchronized锁定的是当前调用方法对象,而Spring AOP 处理事务会生成一个代理对象，而开始事务和提交事务不在synchronized锁定范围内，这就导致在业务逻辑执行后提交事务之前，新的请求线程读取到事务之前的数据，产生了脏读。  
* 使用redis锁失效的原因： 和synchronized类似，释放redis锁的逻辑是在业务逻辑处理完成后，但是没有在提交事务后导致了脏读。  

## 解决办法  
最直接的方法就是把当日预发数量存放在redis中，这样不仅能解决并非产生的脏数据问题，还能提高响应速度  