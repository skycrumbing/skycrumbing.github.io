---
layout: post
title: nginx部署个人博客
categories: webpage
description: nginx部署个人博客
---
将github托管的基于jekyll的博客网站在本地编译成静态网页并放到nginx服务器  

<!-- more -->

## 准备工作  
1，在windows下安装jekyll及相关环境  
2，下载能在windows下运行的nginx  
## 步骤  
1，将github上的网站项目克隆到本地  
2，查看项目路径下的config.yml是否有以下两行配置，如果没有则添加进去  
```
	# source参数填写项目路径，destination填写编译后生成的静态文件存放的目录
	source:       .
	destination:  ./_site
```  
3，在项目路径下打开cmd，运行bundle exec jekyll server,若失败请查看相关环境和依赖是否配置完成  
4，命令执行成功后会有提示，可以直接在本地运行项目，同时在项目路径下生成_site文件夹，里面是编译成功的静态文件
5，可以将_site移动到任何一个路径下，如c:/blog/_site  
6，打开nginx的配置文件nginx.conf，添加如下配置
```
	server {
			listen       80;
			server_name  localhost;
			location /{
				root    c:/blog/_site;
					 index index.html;
			}
			 location ~* ^.+\.(jpg|jpeg|gif|png|ico|css|js|pdf|txt){
				 root c:/blog/_site;
			   }
		}
```
7,运行nginx,浏览器访问http://127.0.0.1

