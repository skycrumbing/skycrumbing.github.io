---
layout: post
title: docker+jenkins+mysql+java
tags:
- io
categories: server
description: docker+jenkins+mysql+java
---
## 持续集成
今天使用docker容器部署jenkins和mysql，再使用jenkins部署springboot应用。  
<!-- more -->

## 物理机jdk下载  
物理机jdk下载并将其解压放置到 /home/jdk/jdk1.8.0_141目录下。目前为止可行的wget下载命令  
```
wget --no-cookies --no-check-certificate --header "Cookie: gpw_e24=http%3A%2F%2Fwww.oracle.com%2F; oraclelicense=accept-securebackup-cookie" "http://download.oracle.com/otn-pub/java/jdk/8u141-b15/336fa29ff2bb4ef291e347e091f7f4a7/jdk-8u141-linux-x64.tar.gz"
```  
## 物理机maven下载
物理机maven下载并解压到/usr/local/maven/apache-maven-3.6.1  
## mysql容器部署  
这里容器初始化时主要需要配置mysql的初始化密码和挂载的目录  
```
docker run -d -p 3306:3306 -e MYSQL_ROOT_PASSWORD=admin -v /home/docker/mysql/conf:/etc/mysql/conf.d -v /home/docker/mysql/data:/var/lib/mysql -v /home/docker/mysql/logs:/logs --name mysql mysql:5.6
```
## jenkins部署   
### jenkins容器下载并运行  
```
docker run -d -p 18080:8080 -p 50000:50000 --restart=always --privileged=true -v /home/jenkins:/var/jenkins_home -v  /usr/local/maven/apache-maven-3.6.1:/usr/share/apache-maven -v /home/maven/local-Repository:/home/maven/local-Repository -v  /home/jdk/jdk1.8.0_141:/usr/share/jdk1.8.0_141 -e JAVA_OPTS=-Duser.timezone=Asia/Shanghai --name jenkins jenkins/jenkins
```  
**参数解读**  
-d：后台运行镜像  
-p：18080:8080 -p 50000:50000：将容器的8080端口映射到服务器的18080端口，容器的50000端口映射到服务器的50000端口以进行通信。  
--restart=always：容器退出时将自动重启该容器，确保Jenkins容器一直运行。  
--privileged=true：container内的root用户拥有真正的root权限。  
-v /home/jenkins:/var/jenkins_home：将硬盘上/home/jenkins挂载到容器的/var/jenkins_home目录，作为jenkins的工作目录。  
/usr/local/maven/apache-maven-3.6.1:/usr/share/apache-maven：maven配置目录映射。  
-v /usr/local/maven/apache-maven-3.6.1:/usr/share/apache-maven -v /home/maven/local-Repository：maven仓库映射。  
-v /home/jdk/jdk1.8.0_141:/usr/share/jdk1.8.0_141：java jdk映射。  
-e JAVA_OPTS=-Duser.timezone=Asia/Shanghai --name jenkins jenkins/jenkins：设置jenkins的时区为上海。  
**注意事项**  
各个挂载目录的权限最好设置为其他用户可读可写可执行  
### 进入Jenkins web管理界面进行配置  
输入ip:18080就可以进入到Jenkins  
- 初次进入需要通过管理员密码，可以在物理机通过下面命令查看密码：  
```
 cat /home/jenkins/secrets/initialAdminPassword
```  
- 然后我们安装推荐插件。  
- 创建用户，之后我们就可以通过目录进入管理界面。  
- 接下来配置全局工具，这里主要配置jdk和maven，git已经在推荐插件里安装过了。  
![全局配置](\assets\img\docker_jenkins_1.png)  
![全局配置](\assets\img\docker_jenkins_2.png)  
![全局配置](\assets\img\docker_jenkins_3.png)  
最后保存配置。  
- 配置完成后就可以开始新建任务了，选择构建一个自由风格的软件项目  
源码管理选择git配置项目的github信息  
构建步骤中添加调用顶层 Maven 目标,版本我们选择之前配置的。目标就是maven需要执行的命令，这里我们填clean install  
![maven配置](\assets\img\docker_jenkins_4.png)  
最后保存配置
- 点击立即构建成功后会在本地仓库中生成项目对应的jar包，在本次示例中仓库目录为
/home/maven/local-Repository







