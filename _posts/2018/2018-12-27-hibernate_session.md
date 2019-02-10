---
layout: post
title: 关于hibernate的使用问题
tags:
categories: hibernate
description: 一级缓存的使用问题
---
## 场景
在一个事务没有提交之前先进行insert()和save()操作，然后再查询刚插入的这条数据，数据库表设置的默认值不生效。

      <!-- more -->

## 原因
在事务提交前save()操作会保存到一级缓存session中而不更新到数据库，再使用Hibernate查询对象的时候会先查询session中的数据，有就直接取出，没有才去数据库中查询。因为事务没有提交，数据没有更新到数据库，session也没有刷新，所以取出的数据就是你插入的数据，数据库表设置的默认值不生效
