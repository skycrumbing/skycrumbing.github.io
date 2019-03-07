---
layout: post
title: Vue脚手架项目打包成桌面应用
tags:
- vue
- js
categories: webpage
description: Vue脚手架项目打包成桌面应用
---
本篇内容基于前面所讲的部分：Vue脚手架项目打包。即已经在项目路径下生成dist文件夹的基础上打包成exe桌面程序。使用的是基于Electron的打包工具Electron-packager。  

<!-- more -->

## Electron  
Electron就是可以让你用Javascript、HTML、CSS来编写运行于Windows、macOS、Linux系统之上的桌面应用的库。  
**使用方式**：  
1，首先快速创建一个官方示例的  
```
	git clone https://github.com/electron/electron-quick-start
	cd electron-quick-start
	npm install
	npm start
```
2，创建成功后打开项目文件夹，复制里面的main.js到vue项目的build文件夹中，并改名为electron.js，如下：  
![build文件夹结构](\assets\img\vue_pack2exe_1.jpg)  
3，打开该文件，更改两个地方（下标①，②标注），配置如下：  
```
	// Modules to control application life and create native browser window
	const {app, BrowserWindow} = require('electron')
	//①要添加的部分
	const path = require('path')
	const url = require('url')

	// Keep a global reference of the window object, if you don't, the window will
	// be closed automatically when the JavaScript object is garbage collected.
	let mainWindow

	function createWindow () {
	  // Create the browser window.
	  mainWindow = new BrowserWindow({
		width: 800,
		height: 600,
		webPreferences: {
		  nodeIntegration: true
		}
	  })

	  //②这个无法使用，需要用下面的方法替代
	  // and load the index.html of the app.
	  //mainWindow.loadFile(''index.html'')
	  //替代的部分
	  mainWindow.loadURL(url.format({
		pathname: path.join(__dirname, '../dist/index.html'),
		protocol: 'file:',
		slashes: true
	  }))

	  // Open the DevTools.
	  // mainWindow.webContents.openDevTools()

	  // Emitted when the window is closed.
	  mainWindow.on('closed', function () {
		// Dereference the window object, usually you would store windows
		// in an array if your app supports multi windows, this is the time
		// when you should delete the corresponding element.
		mainWindow = null
	  })
	}

	// This method will be called when Electron has finished
	// initialization and is ready to create browser windows.
	// Some APIs can only be used after this event occurs.
	app.on('ready', createWindow)

	// Quit when all windows are closed.
	app.on('window-all-closed', function () {
	  // On macOS it is common for applications and their menu bar
	  // to stay active until the user quits explicitly with Cmd + Q
	  if (process.platform !== 'darwin') {
		app.quit()
	  }
	})

	app.on('activate', function () {
	  // On macOS it's common to re-create a window in the app when the
	  // dock icon is clicked and there are no other windows open.
	  if (mainWindow === null) {
		createWindow()
	  }
	})

	// In this file you can include the rest of your app's specific main process
	// code. You can also put them in separate files and require them here.
```
4，在项目的package.json中添加开发工具的模块依赖，并使用cnpm install命令进行下载安装  
```
	"devDependencies": {
		"electron": "^4.0.7",
		"electron-packager": "^13.1.1",
		……
	}
```
5，安装成功后在package.json中添加脚本命令  
```
"scripts": {
	"electron_build": "electron-packager ./dist/ ISR --platform=win32 --arch=x64  --overwrite"
	……
}

```
6，在package.json中添加项目入口信息，如下：  
![项目入口信息](\assets\img\vue_pack2exe_2.jpg)  
7，将刚才的electron.js，package.json复制到已经打包成静态文件的dist目录下，并修改electron.js的路径信息  
```
	mainWindow.loadURL(url.format({
		pathname: path.join(__dirname, './index.html'),
		protocol: 'file:',
		slashes: true
	  }))
```
8，运行npm run electron_dev，在当前项目下就会生成一个ISR-win32-x64文件夹，打开文件夹，运行ISR.exe即可
