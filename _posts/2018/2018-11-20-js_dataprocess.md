---
layout: post
title: js数据处理
tags:
- js
categories: webpage
description: js数据处理
---
## js数据处理
最近的工作中涉及到前端，需要对后台的数据进行处理，由于对js不是很熟悉，每次写代码时都需要百度，所以将常用的函数进行了总结。
<!-- more -->

## 数组操作
-  **js判断一个字符串是以某个字符串开头**  
```
	var fdStart = strCode.indexOf("ssss");
	if(fdStart == 0){
  	表示strCode是以ssss开头；
	}else if(fdStart == -1){
  	表示strCode不存在ssss字符串
	}else if(fdStart > 0){
  	表示strCode包含sss字符串
	}
```
-  **js替换或者删除一个或多个元素** 

1，splice(index,len,[item]):该方法会改变原始数组。  
index:数组开始下标 len: 替换/删除的长度 item:替换的值，删除操作的话 item为空  
删除  
```
	var arr2 = ['a','b','c','d'] 
	arr2.splice(1,2); 
	console.log(arr2); 
	//['a','d'] 
```
替换  
```
	var arr2 = ['a','b','c','d']; 
	arr2.splice(1,2,'ttt'); 
	console.log(arr2);  
	//['a','ttt','d'] 替换起始下标为1，长度为2的两个值为‘ttt'，len设置的1 
```
添加
```
	var arr = ['a','b','c','d']; 
	arr.splice(1,0,'ttt'); 
	console.log(arr);  
	//['a','ttt','b','c','d'] 表示在下标为1处添加一项'ttt'
```
2，delete方法删除掉数组中的元素后，会把该下标出的值置为undefined,数组的长度不会变  
```
var arr = ['a','b','c','d']; 
delete arr[1]; 
arr; 
//["a", undefined, "c", "d"] 中间出现两个逗号，数组长度不变，有一项为undefined  
```
## js整数取整
-  **丢弃小数部分,保留整数部分**  
```
	parseInt(7/2) 
```
-  **向上取整,有小数就整数部分加1**  
```
	 Math.ceil(7/2)  
```
-  **四舍五入**  
```
	Math.round(7/2) 
```
-  **向下取整，丢弃小数部分**  
```
	Math.floor(7/2) 
```
## js时间操作
```
	<script type="text/javascript">

		var d=new Date()
		var day=d.getDate()
		var month=d.getMonth() + 1
		var year=d.getFullYear()

		document.write(day + "." + month + "." + year)
		document.write(year + "/" + month + "/" + day)

	</script>
```
结果：  
31.1.2019  
2019/1/31  





















