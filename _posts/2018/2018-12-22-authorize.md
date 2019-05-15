---
layout: post
title: 第三方授权
tags:
- authorize
categories: thinking
description: 第三方授权
---
## 第三方授权
一个应用要访问另外一个应用的资源， 该如何授权？  
比如：  
1. 微信公众平台的网站要通过微信获取关注者的个人信息   
2. 一个信用卡管家的网站要读取用户网易邮箱的信息  

      <!-- more -->
      
下面以信用卡管家的网站要读取用户网易邮箱的信息为例展示三种授权方式

## Resource Owner Password Credentials Grant  
资源所有者密码凭据许可  
信用卡管家网站读取用户网易邮箱信息，需要用户提供用户名和密码，程序通过用户名和密码登陆访问。  
问题：十分不安全。用户信息容易被窃取  
## Implicit Grant  
隐式许可  
1. 信用卡管家网站向网易申请授权，网易给网站提供app_id 和app_secret。  
2. 为用户提供一个入口，用户点击之后会重定向到网易的认证系统去登陆（当然要携带网易提供的app_id和app_secret，网易才知道你的网站已经被授权），认证系统会让用户输入用户名和密码并提示你是否允许该网站访问你的邮箱信息，确认以后会再次重定向到信用卡管家网站，同时带一个“token”过来。  
3. 用这个“token”访问网易的API即可获得用户的信息。    

注意事项：这个token是用hash mark（#）进行标识返回的，表示它是一个fragment。 

**Fragment的特性：** 
- 有别于 ?，? 后面的查询字符串会被网络请求带上服务器，而 fragment 不会被发送的服务器；  
- fragment 的改变不会触发浏览器刷新页面，但是会生成浏览历史；  
- fragment 会被浏览器根据文件媒体类型（MIME type）进行对应的处理；
- Google 的搜索引擎会忽略 # 及其后面的字符串。  

**使用Fragment的目的：** 只有Javascript 能访问它，并且它不会再次通过http request 发到别的服务器， 这是为了提高安全性  
信用卡管家要获取用户网易邮箱的信息为例展示隐私许可步骤    
![隐式许可](\assets\img\authorize_1.jpg)  
**问题：** 这个token 以明文的方式发送给了用户的浏览器， 虽然是https ，不会被别人窃取，可是浏览器的历史记录或者访问日志中就能找到还是存在安全问题。
## Authorization Code Grant  
授权码许可  
和之前的思路类似，只是引用了一个授权码(authorization code)，当用户用网易账号登录的时候， 网易认证中心这一次不直接发token,而是发一个授权码(authorization code)，信用卡管家服务器端获得authorization code，在后台再次访问网易认证中心，这次才能获得真正的token。  
注意：授权码和信用卡管家申请的app_id，app_secret关联， 只有信用卡管家发出的token请求， 网易认证中心才认为合法； 为了安全性，还可以让授权码有时间限制，比如5分钟失效，还有可以让授权码只能换一次token, 第二次就不行了。  
![授权码许可](\assets\img\authorize_2.jpg)  



