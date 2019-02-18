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
### js判断一个字符串是否包含某个字符串    
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
### js替换或者删除一个或多个元素  

**splice(index,len,[item]):该方法会改变原始数组长度。**  
index:数组开始下标  
len: 替换/删除的长度  
item:替换的值，删除操作的话 item为空    
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
```html
	var arr = ['a','b','c','d']; 
	arr.splice(1,0,'ttt'); 
	console.log(arr);  
	//['a','ttt','b','c','d'] 表示在下标为1处添加一项'ttt'
```
**delete：删除掉数组中的元素后，会把该下标出的值置为undefined,数组的长度不会变** 
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
```html
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
## ES6新方法  
### 判断字符串是否包含字符字串
**includes()：返回布尔值，表示是否找到了参数字符串。**  
**startsWith()：返回布尔值，表示参数字符串是否在原字符串的头部。**  
**endsWith()：返回布尔值，表示参数字符串是否在原字符串的尾部。**  
```
	let s = 'Hello world!';

	s.startsWith('Hello') // true
	s.endsWith('!') // true
	s.includes('o') // true
```  
这三个方法都支持第二个参数，表示开始搜索的位置。  
```
	let s = 'Hello world!';

	s.startsWith('world', 6) // true
	s.endsWith('Hello', 5) // true
	s.includes('Hello', 6) // false
```
上面代码表示，使用第二个参数n时，endsWith的行为与其他两个方法有所不同。它针对前n个字符，而其他两个方法针对从第n个位置直到字符串结束。  
### repeat()  
repeat方法返回一个新字符串，表示将原字符串重复n次。  
```
	'x'.repeat(3) // "xxx"
	'hello'.repeat(2) // "hellohello"
	'na'.repeat(0) // ""
```
参数如果是小数，会被取整。  
```
	'na'.repeat(2.9) // "nana"
```
如果repeat的参数是负数或者Infinity，会报错。  
```
	'na'.repeat(Infinity)
	// RangeError
	'na'.repeat(-1)
	// RangeError 
```
但是，如果参数是 0 到-1 之间的小数，则等同于 0，这是因为会先进行取整运算。0 到-1 之间的小数，取整以后等于-0，repeat视同为 0。  
```
	'na'.repeat(-0.9) // ""
```
参数NaN等同于 0。  
```
	'na'.repeat(NaN) // ""
```
如果repeat的参数是字符串，则会先转换成数字。  
```
	'na'.repeat('na') // ""
	'na'.repeat('3') // "nanana"
```
### padStart()，padEnd()  
ES2017 引入了字符串补全长度的功能。如果某个字符串不够指定长度，会在头部或尾部补全。padStart()用于头部补全，padEnd()用于尾部补全  
```
	'x'.padStart(5, 'ab') // 'ababx'
	'x'.padStart(4, 'ab') // 'abax'

	'x'.padEnd(5, 'ab') // 'xabab'
	'x'.padEnd(4, 'ab') // 'xaba'
```
上面代码中，padStart()和padEnd()一共接受两个参数，第一个参数是字符串补全生效的最大长度，第二个参数是用来补全的字符串。  
如果原字符串的长度，等于或大于最大长度，则字符串补全不生效，返回原字符串。  
```
	'xxx'.padStart(2, 'ab') // 'xxx'
	'xxx'.padEnd(2, 'ab') // 'xxx'
```
如果用来补全的字符串与原字符串，两者的长度之和超过了最大长度，则会截去超出位数的补全字符串。   
```
	'abc'.padStart(10, '0123456789')
	// '0123456abc'
```
如果省略第二个参数，默认使用空格补全长度。  
```
	'x'.padStart(4) // '   x'
	'x'.padEnd(4) // 'x   '
```
padStart()的常见用途是为数值补全指定位数。下面代码生成 10 位的数值字符串。  
```
	'1'.padStart(10, '0') // "0000000001"
	'12'.padStart(10, '0') // "0000000012"
	'123456'.padStart(10, '0') // "0000123456"
```  
另一个用途是提示字符串格式。  
```
	'12'.padStart(10, 'YYYY-MM-DD') // "YYYY-MM-12"
	'09-12'.padStart(10, 'YYYY-MM-DD') // "YYYY-09-12"
```





















