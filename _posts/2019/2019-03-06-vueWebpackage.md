---
layout: post
title: Vue脚手架项目打包
tags:
- vue
- js
categories: webpage
description: Vue脚手架项目打包
---
使用vue脚手架创建，开发项目后，我们可以通过npm run build 将项目打包成静态文件直接在浏览器中打开。打包后的结构如下。

<!-- more -->

![webpackage结构](\assets\img\vue_webpackage_1.jpg)  
## 静态资源的引入问题  
将index.html用浏览器打开，却没有任何内容，用编辑器打开后发现所有资源引入的相对路径都是以/static/开头的，如下：  
src=/static/js/vendor.78b0c67da9f7ca9ebf03.js  
而正确路径应该是以./static/开头。  
**解决方法**：  
将项目下的config/index.js文件中build->assetsPublicPath的'/'改为'./'。  
如下：  
![资源路径解决办法](\assets\img\vue_webpackage_2.jpg)  
## 字体图标的引入问题  
静态资源引入问题解决之后用浏览器打开index.html，结果图标字体依旧无法显示，在浏览器控制台发现这些字体图标的相对路径都是以./static/css/static/fonts开头。  
而正确路径应该是以./static/fonts开头。  
**解决方法**：  
在项目下的build/utils.js添加publicPath: '../../'  
如下：  
![子体图标路径解决办法](\assets\img\vue_webpackage_3.jpg)  

