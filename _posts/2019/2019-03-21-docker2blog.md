---
layout: post
title: docker部署个人博客
tags:
- docker
- nginx
categories: server
description: docker部署个人博客
---
制作一个docker容器部署github托管的基于jekyll的博客网站。  

<!-- more -->

## 准备工作  
1，一个Linux服务器  
2，在服务器中安装docker，并且配置镜像加速器，启动docker  
## 步骤  
### 创建ubuntu容器，运行并进入容器内的shell  
```
sudo docker run -i -t ubuntu /bin/bash  
```
如果存在就进入容器，不存在就会从docker仓库下载。  
### 安装ruby和 Bundler和 Jekyll  
在这里遇到了问题:  
在使用bundler install命令安装Jekyll时报错  
```
An error occurred while installing nokogiri (x.x.x), and Bundler cannot continue
```
解决办法：  
使用如下命令，然后再bundler install  
```
sudo apt-get install build-essential patch ruby-dev zlib1g-dev liblzma-dev
```
### 安装git
安装git,创建git项目文件夹,将github的项目克隆到文件夹下  
### 生成静态文件  
在项目路径下运行bundle exec jekyll server  
命令执行成功后会有提示，同时在项目路径下生成\_site文件夹，里面是编译成功的静态文件，ctrl+c退出运行状态  
在这个编译环节出现了问题：  
容器内的ubuntu系统并没有使用utf-8的字符集  
使用locale命令查看系统正在使用的编码方式  
```
LANG=
LANGUAGE=
LC_CTYPE="POSIX"
LC_NUMERIC="POSIX"
LC_TIME="POSIX"
LC_COLLATE="POSIX"
LC_MONETARY="POSIX"
LC_MESSAGES="POSIX"
LC_PAPER="POSIX"
LC_NAME="POSIX"
LC_ADDRESS="POSIX"
LC_TELEPHONE="POSIX"
LC_MEASUREMENT="POSIX"
LC_IDENTIFICATION="POSIX"
LC_ALL=
```
使用locale -a查看系统支持的编码方式  
```
C
C.UTF-8
POSIX
```
解决方式：  
安装vim，使用vim编辑 /etc/profile。将“export LANG="C.UTF-8”命令添加在profile最后  
然后使用source /etc/profile刷新  
注意：一旦从容器中退出，然后再进去，编码集还原，需要再次使用source /etc/profile再次刷新才可   
### 安装nginx以及相关依赖的包  
安装成功后会在/usr/local下生成nginx文件夹  
使用vim编辑/usr/local/nginx/conf/nginx.conf。更改nginx.conf的server配置  
```
server {
        listen       4000; #监听端口
        server_name  47.106.248.123; #服务器ip 

        #charset koi8-r;

        #access_log  logs/host.access.log  main;
		#项目路径
        location / {
            root    /myblog/data/skycrumbing.github.io/_site;
	    index  index.html;
	}
	location ~* ^.+\.(jpg|jpeg|gif|png|ico|css|js|pdf|txt){
				 root /myblog/data/skycrumbing.github.io/_site;
	}
}
```
检查nginx配置是否有问题  
```
/usr/local/nginx/sbin/nginx -t -c /usr/local/nginx/conf/nginx.conf
```  
如果没问题  
```
nginx: the configuration file /usr/local/nginx/conf/nginx.conf syntax is ok
nginx: configuration file /usr/local/nginx/conf/nginx.conf test is successful
```
到现在容器的环境基本搭建成功  
### 退出容器  
```
exit
```
### 将容器转化为镜像  
```
docker commit ubuntu skycrumbing/myblog  
```
转化成功后查看本地镜像  
```
docker images
```  
### 运行镜像  
```
docker run -dit --privileged -p4000:4000 -p80:80 --name myblog skycrumbing/myblog
```  
run的各项参数：  
-dit 是 -d -i -t 的缩写。-d表示detach，即在后台运行。-i表示提供交互接口这样才可以通过docker和跑起来的操作系统交互。-t表示提供一个 tty (伪终端)，与-i配合就可以通过ssh 工具连接到这个容器里面去了  
-p4000:4000。第一个4000，表示在服务器上开放4000端口。 第二个4000表示在容器里开放4000端口。 这样当访问服务器的21端口的时候，就会间接地访问到容器里了  
-p80:80。也是同样的道理。
如果运行失败可能是容器已经存在没有删除，所以需要先删除  
```
docker rm myblog
```
### 进入容器  
```
docker exec -it myblog /bin/bash
```
### 启动nginx  
```
/usr/local/nginx/sbin/nginx -c /usr/local/nginx/conf/nginx.conf
```
### 退出容器
使用“exit”退出容器，由于运行镜像的时候已经设置了后台运行的参数，所以退出容器后依然在后台运行。  
### 开启阿里云端口  
如果是阿里云的服务器需要开启对外的端口。  
由于我需要使用的是4000端口和容器进行交互，所以需要开启4000端口。  
### 访问测试  
浏览器访问网站:http://47.106.248.123:4000  
### 上传镜像到仓库    





