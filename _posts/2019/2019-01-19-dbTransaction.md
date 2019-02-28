---
layout: post
title: 数据库事务
tags:
- spring
categories: database
description: 数据库事务
---
## 事务机制（ACID）
**原子性**： 事务是最小的执行单位，不允许分割。事务的原子性确保动作要么全部完成，要么完全不执行；  
**一致性**： 执行事务前后，数据保持一致；  
**隔离性**： 并发访问数据库时，一个用户的事物不被其他事物所干扰，各并发事务之间数据库是独立的；  
**持久性**: 一个事务被提交之后。它对数据库中数据的改变是持久的，即使数据库发生故障也不应该对其有任何影响。  

<!-- more -->

## 事务隔离级别  
由于事务的并发很难保证事务的特性（ACID）， 事务的并发会带来四种问题：  
**脏读**：一个事务访问了另一个事务未提交的数据  
**丢失修改**：两个事务并发进行修改操作导致先提交的事务修改数据丢失  
如事务TA和TB同时修改C的数据：  
1,TA读C  
2,TB读C  
3,TA修改C然后提交事务  
4,TB修改C然后提交事务  
这就导致A的提交被覆盖  
**不可重复读**：TA事务第一次读取数据之后TB事务修改了数据并提交了，事务TA再次读取数据导致数据不一致。  
**幻读**：TA事务第一次读取数据之后TB事务添加了数据并提交了，事务TA再次读取数据导致数据多了一些内容就像幻觉一样。  
**为了解决事务并发带来的问题，数据库定义了几种不同的事务隔离级别**    
**READ_UNCOMMITTED（未提交读）**: 最低的隔离级别，允许读取尚未提交的数据变更，可能会导致脏读、幻读或不可重复读  
**EAD_COMMITTED（提交读）**: 允许读取并发事务已经提交的数据，可以阻止脏读，但是幻读或不可重复读仍有可能发生  
**REPEATABLE_READ（可重复读）**: 对同一字段的多次读取结果都是一致的，除非数据是被本身事务自己所修改，可以阻止脏读和不可重复读，但幻读仍有可能发生。  
**ERIALIZABLE（串行）**: 最高的隔离级别，完全服从ACID的隔离级别。所有的事务依次逐个执行，这样事务之间就完全不可能产生干扰，也就是说，该级别可以防止脏读、不可重复读以及幻读。但是这将严重影响程序的性能。通常情况下也不会用到该级别。  
Mysql 默认采用的 REPEATABLE_READ隔离级别 Oracle 默认采用的 READ_COMMITTED隔离级别.  
## Spring事务管理接口  
Spring事务管理接口包括三个部分  
PlatformTransactionManager： （平台）事务管理器  
TransactionDefinition： 事务定义信息(事务隔离级别、传播行为、超时、只读、回滚规则)   
TransactionStatus： 事务运行状态  
### PlatformTransactionManager接口  
Spring给数据持久化话框架（hibernate，mybatis，JDBC）提供事务管理器接口，让这些框架去实现事务管理，它定义了三个方法  
获取事务  
提交事务  
回滚事务  
几个常见的接口实现类:  
![事务实现类](\assets\img\dbTransaction_1.png)  
### TransactionDefinition接口  
这个接口主要是定义事务属性事务属性包括5个方面:  
隔离级别，传播行为，回滚规则，是否只读，事务超时  
#### Spring事务隔离级别  
**TransactionDefinition.ISOLATION_DEFAULT**: 使用后端数据库默认的隔离级别，Mysql 默认采用的 REPEATABLE_READ隔离级别 Oracle 默认采用的 READ_COMMITTED隔离级别.  
**TransactionDefinition.ISOLATION_READ_UNCOMMITTED**: 最低的隔离级别，对应数据库隔离级别READ_UNCOMMITTED  
**TransactionDefinition.ISOLATION_READ_COMMITTED**: 对应数据库隔离级别READ_UNCOMMITTED  
**TransactionDefinition.ISOLATION_REPEATABLE_READ**: 对应数据库隔离级别REPEATABLE_READ  
**TransactionDefinition.ISOLATION_SERIALIZABLE**: 对应数据库隔离级别ERIALIZABLE  
#### Spring事务传播行为  
当一个事务方法调用另一个事务方法时事务的存在机制  
**支持当前事务的情况**：  
**TransactionDefinition.PROPAGATION_REQUIRED**： 如果当前存在事务，则加入该事务；如果当前没有事务，则创建一个新的事务。  
**TransactionDefinition.PROPAGATION_SUPPORTS**： 如果当前存在事务，则加入该事务；如果当前没有事务，则以非事务的方式继续运行。  
**TransactionDefinition.PROPAGATION_MANDATORY**： 如果当前存在事务，则加入该事务；如果当前没有事务，则抛出异常。  
    
**不支持当前事务的情况：**  
**TransactionDefinition.PROPAGATION_REQUIRES_NEW**： 创建一个新的事务，如果当前存在事务，则把当前事务挂起。  
**TransactionDefinition.PROPAGATION_NOT_SUPPORTED**： 以非事务方式运行，如果当前存在事务，则把当前事务挂起。  
**TransactionDefinition.PROPAGATION_NEVER**： 以非事务方式运行，如果当前存在事务，则抛出异常。  
    
**事务嵌套情况：**  
**TransactionDefinition.PROPAGATION_NESTED**： 如果当前存在事务，则创建一个事务作为当前事务的嵌套事务来运行；如果当前没有事务，则该取值等价于TransactionDefinition.PROPAGATION_REQUIRED。外部事务的回滚也会导致嵌套子事务的回滚。内层事务操作失败并不会引起外层事务的回滚。  
#### 事务超时属性  
一个事务允许执行的最长时间，超时则回滚。  
#### 事务只读属性  
对事物资源是否执行只读操作，进行只读操作会提高性能。  
#### 回滚规则  
义事务回滚规则。定义了哪些异常会导致事务回滚而哪些不会。默认情况下，事务只有遇到运行期异常时才会回滚，而在遇到检查型异常时不会回滚  
### TransactionStatus接口  
TransactionStatus接口用来记录事务的状态 该接口定义了一组方法,用来获取或判断事务的相应状态信息。  
```
	public interface TransactionStatus{
		boolean isNewTransaction(); // 是否是新的事物
		boolean hasSavepoint(); // 是否有恢复点
		void setRollbackOnly();  // 设置为只回滚
		boolean isRollbackOnly(); // 是否为只回滚
		boolean isCompleted; // 是否已完成
	} 
```





