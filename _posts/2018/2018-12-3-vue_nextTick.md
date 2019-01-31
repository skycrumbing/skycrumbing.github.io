---
layout: post
title: vue之$nextTick()
tags:
- vue
categories: webpage
description: vue之$nextTick()
---
## Vue.$nextTick()
Vue 实现响应式并不是数据发生变化之后 DOM 立即变化，而是按一定的策略进行 DOM 的更新。  
Vue.$nextTick(callback)，当dom发生变化更新后执行的回调。

<!-- more -->

## 代码
```
	<template>
	  <div class="hello">
		<div ref="msgDiv">{{msg}}</div>
		<div>setTimeout: {{msg1}}</div>
		<div>$nextTick: {{msg2}}</div>
		<div>正常情况: {{msg3}}</div>
		<button @click="changeMsg">BUTTON</button>
	   </div>
	</template>

	<script>
	export default {
	  name: 'HelloWorld',
	  data () {
		return {
		  msg: "VUE",
		  msg1: '',
		  msg2: '',
		  msg3: '',
		}
	  },
	  methods: {
		changeMsg() {
		  this.msg = "Hello world."
		  setTimeout(() => {
			this.msg1 = this.$refs.msgDiv.innerHTML // Hello World.
		  }, 0);
		  this.$nextTick(() => {
			this.msg2 = this.$refs.msgDiv.innerHTML // Hello World.
		  });
		  this.msg3 = this.$refs.msgDiv.innerHTML // Vue.(没有改变)
		},
	  }

	}
	</script>
```
点击前  
![点击前](\assets\img\vue_nextTick.jpg)
点击后  
![点击后](\assets\img\vue_nextTick1.jpg)

