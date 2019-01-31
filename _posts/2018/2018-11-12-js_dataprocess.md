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


























