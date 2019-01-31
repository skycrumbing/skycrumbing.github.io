---
layout: post
title: vue组件传值
tags:
- vue
categories: webpage
description: vue组件传值
---
## vue组件
组件之间的关系：父组件是封装子组件的，即父组件包含子组件。
<!-- more -->

## 父组件传值给子组件
父组件通过子组件设置的props给子组件传值。  
## 子组件给父组件传值  
父组件可以通过v-on来监听一个自定义事件eventName。  
子组件可以通过$emit(eventName, optionalPayload)触发父组件的自定义事件，并且可以将optionalPayload传递给父组件自定义事件的函数。  
## 代码
### 父组件给子组件传值
子组件ChildT  
```
	<template>
		<div>
			<p>从父组件得到3:{ {childMessage } }/</p>
		</div>
	</template>

	<script>
		export default{
		  props:['childMessage'],
		  data(){
			  return{

			  }
		  },
		}
	</script>
```
父组件ParentT
```
	<template>
		<div>
		  <h4>父组件向子组件传值</h4>
		  <input v-model="message"/>
		  <child-t :childMessage="message"></child-t>
		</div>
	</template>

	<style>

	</style>

	<script>
	  import ChildT from './ChildT.vue'
		export default{
		  components:{
				ChildT
		  },
		  data(){
			  return {
				  message:'hello'
			  }
		  }
		}
	</script>
```
### 子组件给父组件传值
子组件ChildToParent  
```
	<template>
		<div>
		  <h4>子组件向父组件传递值</h4>
		  <div>父组件传递过来的:{{getFromP}}</div>
		  <input type="text" v-model="sendToPData"/>
		  <!--点击事件-->
		  <button @click="sendMsgToParent">向父组件传值</button>
		</div>
	</template>

	<style>

	</style>

	<script>
		export default{
			data () {
				return {
				  sendToPData:''
				}
			},
			props:['getFromP'],
			methods:{
	//          响应该点击事件的函数中使用$emit来触发一个自定义事件，并传递一个参数
				sendMsgToParent:function () {
				   this.$emit('listenToChildEvent',this.sendToPData)
				}
			}
		}
	</script>
```
父组件ParentGetFromChild  
```
	<template>
		<div>
			<div id="app">
			  <!--父组件中的子标签中监听该自定义事件并添加一个响应该事件的处理方法-->
			  <child-to-p :getFromP="parentMsg" @listenToChildEvent="showMsgFromChild"></child-to-p>
			  <div>从子组件传递过来的值:{{dataFromChild}}</div>
			</div>
		</div>
	</template>

	<style>

	</style>

	<script>
	  import ChildToP from './ChildToParent.vue';
		export default{
			data () {
				return {
				  parentMsg:'hello child',
				  dataFromChild:''
				}
			},
		  components:{
				ChildToP
		  },
		  methods:{
				showMsgFromChild:function (data) {
				  console.log('从子组件传递过来的:'+data)
				  this.dataFromChild = data;
				}
		  }
		}
	</script>
```
### 页面 
```
	<template>
		<div>
			<parent-t></parent-t>
			<parent-get-from-child></parent-get-from-child>
		</div>
	</template>

	<style>

	</style>

	<script>
	  import ParentT from  './ParentT.vue';
	  import ParentGetFromChild from './ParentGetFromChild.vue';
		export default{
			data () {
				return {}
			},
		  name:'Props',
		  components:{
			ParentT,
			ParentGetFromChild
		  }
		}
	</script>
```
## 结果  
![结果](\assets\img\vue_component.jpg)

























