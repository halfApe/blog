---
title: 运维篇Docker之灵魂拷问2-1
---

上篇博文中，我们已经对Docker有了基本的了解，这次我们针对Docker的两个核心组件进行灵魂拷问,由于篇幅的关系，我们本节主要拷问Images，至于Containers，留至下半节进行拷问。

## Images

在上节的最后，我们提到了Docker设计猜想，这次我们以tomcat为例，来一步一步证明我们的猜想

![](https://raw.githubusercontent.com/Alvin33/images/master/Docker-image.png)

从上节内容我们已经知道，在执行了dokcer pull tomcat时，我们会从hub.docker.com去寻找tomcat image。

### Image是如何产生的

> 灵魂拷问1:这个image是怎么来的呢?

我们先来打开一个地址:

>  https://github.com/docker-library/tomcat/tree/master/8.5/jdk8/openjdk 

从这个地址，我们可以发现：image都是由一个Dockerfile得来的,不信？可以搜搜其他的仓库~

打开Dockerfile,我们发现它是长这个样子的,篇长截片（由于篇幅太长，我们只截图部分片段）,看起来很像是一些固定的语法，有木有~

```dockerfile
FROM openjdk:8-jdk
ENV CATALINA_HOME /usr/local/tomcat
ENV PATH $CATALINA_HOME/bin:$PATH
RUN mkdir -p "$CATALINA_HOME"
WORKDIR $CATALINA_HOME
......
RUN set -eux; \
	\
	......
	chmod -R +rX .; \
	chmod 777 logs temp work
# verify Tomcat Native is working properly
RUN ......
EXPOSE 8080
CMD ["catalina.sh", "run"]
```

有了Dockerfile，我们来验证一把，他是不是能够生成image。

```shell
#1.查看现有images
docker images
#2.将上述的DOckerfiler文件上传到当前目录
#3.执行生成image操作
docker build -t my-mysql-image .
#4.再次执行查看image操作，就可以发现生成了镜像my-mysql-image
```

上述操作，我们可以得出一个结论：`image都是由Dockerfile得来的`，我们是不是可以写一份Dockerfile来生成自己的image呢？答案是肯定的，要写自己的Dockerfiler,我们得先来看一下他的基本语法：

```Dockerfile
#1.FROM 指定基础镜像
FROM ubuntu:14.04
#2.RUN 在镜像内部执行命令，例如安装软件或者配置环境等，换行可以用""
RUN groupadd -r mysql && useradd -r -g mysal mysql
#3.ENV 设置变量的值，后面可以直接使用${MYSQL_MAJOR}，可通过docker run --e key=value修改
ENV MYSQL_MAJOR 5.7 
#4.LABEL设置镜像标签
LABEL email="itcrazy2016@163.com" 
LABEL name="itcrazy2016
#5.VOLUME 指定数据的挂载目录
VOLUME /usr/lib/mysql
#6.COPY 将宿主机的文件复制到镜像内，如果目录不存在，会自动创建所需要的目录。注意只是复制，不会提取和解压
COPY docker-entrypoint.sh /usr/local/bin/
#7.ADD 将主机的文件复制到镜像内，和COPY类似，只是ADD会对压缩文件提取和解压
ADD application.yml /etc/itcrazy2016/
#8.WORKDIR 指定镜像的工作目录，之后的命令都是基于此目录工作，若不存在则创建
WORKDIR /usr/local
WORKDIR tomcat
RUN touch test.txt
	#会在/usr/local/tomcat下创建test.txt文件
	WORKDIR /root
    ADD app.yml test/
	#会在/root/test下多出一个app.yml文件
#9.CMD 容器启动的时候默认执行命令，若有多个CMD命令，只有最后一个会生效
CMD ["mysqld"] 
或
CMD mysqld
#10.ENTRYPOINT 和CMD的使用类似,与之不同的是：docker run执行时，会覆盖CMD的命令，而ENTRYPOINT不会 
ENTRYPOINT ["docker-entrypoint.sh"]
#11.EXPOSE 指定镜像要暴露的端口，启动镜像时，可以使用-p将该端口映射给宿主机
EXPOSE 3306
```

###  创建属于自己的Image

了解了Dockerfile的基本命令之后，那么我们就可以着手创建自己的image了。在这里我们以一个Spring Boot web项目进行演示，具体操作如下：

```shell
#1.创建一个Spring Boot项目,创建一个Controller类，参考代码：
https://github.com/Alvin33/docker-all
#2.对项目进行打包
mvn clean package
#3.在docker环境中新建一个目录'first-dockerfile'
#4.将所打的jar包'docker-all-0.0.1-SNAPSHOT.jar'上传到该目录中(这里我们可以先执行sudo yum install lrzsz，这样就可以直接把文件拖入到虚机中啦)。并在该目录下创建Dockerfile文件，编写Dockerfile文件内容如下:
FROM openjdk:8
MAINTAINER jarluo
LABEL name="dockerfile-demo" version="1.0" author="jarluo"
COPY docker-all-0.0.1-SNAPSHOT.jar dockerfile-image.jar
CMD ["java","-jar","dockerfile-image.jar"]
#5.根据我们上面建立的Dockerfile构建镜像
docker build -t my-docker-image .
#6.基于image创建container
docker run -d --name user01 -p 8888:8080 my-docker-image
	#我们可以通过docker ps 查看我们创建的容器
#7.查看启动日志
docker logs user01
#8.宿主机上访问curl localhost:8888/dockerfile 如果返回结果 说明我们自己定义的Dockerfile正确啦~
#9.此外，我们还可以在启动一个容器
docker run -d --name user02 -p 8889:8080 my-docker-image

```

到这里，相信小书友们对Docker image的理解更为深刻了。但是可能大家又产生了新的疑惑，我自己创建了image，想让所有的互联网码农都能够使用或者是让自己公司内部的同事使用，该怎么做呢？

### 分享自定义Image

> 灵魂拷问2：自定义image怎样分享出去

这里我们将讨论三种方案，相信总有一种适合你~

- 自定义Image上传hub.docker.com

上节内容中，我们知道了，我们所有的iamge都是从hub上进行拉取的，那么我们当然也可以将自己创建的iamge上传到hub咯。

```shell
#1.首先我们要登录hub官网注册属于我们自己的账户，然后进行登录
#2.在docker机器上登录，会要求输入用户名及密码
docker login
#3.将我们自定义image进行重新命名，docker能找到我们官网的账户
docker tag my-docker-image alvin33/my-docker-image 
#4.将我们自定义的iamge推送到hub
docker push alvin33/my-docker-image
```

推送成功后，刷新官网我们自己的主页，就可以看到刚才上传的image啦，那么别人想使用的话，该怎么操作呢？当然就是和其他镜像一样使用咯

```shell
#1.拉取镜像
docker pull alvin33/my-docker-image
#2.创建容器
dokcer run -d --name user01 -p 6666:8080 alvin33/my-docekr-image
```

- 自定义Image上传阿里云

可能有人会问了，有了docker官网仓库，为什么还要用阿里云仓库呢？我们上面演示的只是一个很小的demo，当上传一个非常大的项目时，会比较慢，所以我们就考虑国内服务器阿里云咯,上正餐

```shell
#1.阿里云官网登录自己的阿里云,打开该网址
https://cr.console.aliyun.com/cn-hangzhou/instances/repositories
#2.宿主机登录阿里云docker仓库,参照阿里云官网访问路径https://cr.console.aliyun.com/cn-hangzhou/instances/credentials
sudo docker login --username=alvinos666 registry.cn-hangzhou.aliyuncs.com
#3.阿里云容器镜像服务创建命名空间alvin33 
#4.给自定义image打tag.
sudo docker tag my-docker-image:latest  registry.cn-hangzhou.aliyuncs.com/alvin33/my-docker-image:v1.0
#5.推送自定义镜像到阿里云docker仓库
sudo docker push registry.cn-hangzhou.aliyuncs.com/alvin33/my-docker-image:v1.0
```

操作完上述步骤后，我们再次刷新镜像仓库，就可以看到我们上传的镜像啦。

想要拉取玩玩的话，命令也直接贴给你,毕竟阿里云的这个网址还是蛮长的。

```shell
#镜像拉取
docker pull registry.cn-hangzhou.aliyuncs.com/alvin33/my-docker-image:v1.0
#创建容器
docker run -d --name user01 -p 6661:8080 registry.cn-hangzhou.aliyuncs.com/alvin33/my-docker-image:v1.0
```

- 自定义Image上传私服Docker Harbor

最后，我们来看看假如我们想搭建一个私服的话，该怎么操作吧,我们可以先搭建Docker Harbor的基本环境，具体操作步骤如下：

```shell
#1.下载版本Docker Harbor，以1.7.1版本为例   
https://github.com/goharbor/harbor/releases
#2.检查当前机器是否安装了docker-compose
docker-compose version
	#2.1如果没有安装，则执行以下命令进行安装
	#2.1.1.下载最新版的docker-compose文件
	sudo curl -L https://github.com/docker/compose/releases/download/1.16.1/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose
	#2.1.2.添加可执行权限
	sudo chmod +x /usr/local/bin/docker-compose
	#2.1.3.测试安装结果
	docker-compose --version
		#测试安装结果的时候可能会爆出以下错误
		Cannot open self /usr/local/bin/docker-compose or archive /usr/local/bin/docker-compose.pkg
		#解决方案：
		参见该网址：https://www.cnblogs.com/sixiweb/p/7048914.html
#3.上传并解压
tar -zxvf harbor-online-installer-v1.7.1.tgz
#4.进入到harbor目录，修改harbor.cfg文件，主要是ip地址的修改成当前机器的ip地址。同时也可以看到Harbor的密码，默认是Harbor12345
hostname = 192.168.110.164
#5.安装harbor
sh install.sh
#6.浏览器访问，比如192.168.110.164(个人虚机地址)，输入用户名admin和密码Harbor12345即可。

#tip:
	当发现页面访问不了的时候，可以重启harbor,进入到harbor目录，命令以下命令：
	docker-compose stop
	docker-compose start
	
```

Docker Harbor 的基本环境搭建好了，那么我们怎么来进行推送呢。且看：

```shell
#1.在宿主机上登录Docker Harbor
docker -D login 192.168.110.164
	#报如下错误:
	Error response from daemon: Get https://192.168.110.164/v2/: dial tcp 192.168.110.164:443: connect: connection refused
	#解决方案参考:https://blog.csdn.net/zyl290760647/article/details/83752877
	A:在需要登陆的docker client端修改lib/systemd/system/docker.service文件，在里面修改ExecStart那一行，增加--insecure-registry=192.168.110/164，
	然后重启docker （systemctl daemon-reload    systemctl restart docker）
	B:在harbor服务器端修改 /etc/docker/daemon.json（如果没有这个文件，自己建），增加
	{
 	 "insecure-registries": ["192.168.110.164"]
  	}
  	以上语句注意空格。
	修改后，同样运行 （systemctl daemon-reload    systemctl restart docker）
	出现以下语句，说明成功.
	WARNING! Your password will be stored unencrypted in /root/.docker/config.json.
Configure a credential helper to remove this warning. See
https://docs.docker.com/engine/reference/commandline/login/#credentials-store
Login Succeeded
#2.推送自定义镜像
	#2.1 登录Harbor UI,新建项目harbor-test，并新建用户alvin33，并在新建的项目harbor-test内添加刚才新建的成员用户:alvin33
	#2.2 使用alvin33重新登录账户(输入用户名时请输入alvin33)
	docker login 192.168.110.164
	#2.3 登录完成后我们就可以进行推送啦，具体执行以下两步操作；
docker tag my-docker-image:latest 192.168.110.164/harbor-test/my-docker-image:v1.0
docker push 192.168.110.164/harbor-test/my-docker-image:v1.0
```

至此，我们使用Harbor私服来管理我们的自定义镜像也就完成了。
![](https://raw.githubusercontent.com/Alvin33/images/master/Harbor%E8%87%AA%E5%AE%9A%E4%B9%89%E9%95%9C%E5%83%8F.png)

## 写在最后：

文章的开头 ，我们提到了docker的分层思想，读完文章之后，我们应该能感觉到我们其实一直就是在讲一个东西，那就是Dockerfile。而Dockerfile的命令执行过程，不就如做千层蛋糕一样层层在叠加吗？

下半节，我们将对Containers进行浅析，感受下在images layer之上的 Contaiers layer。

## 说点题外话：

写这篇文章，没有配太多图，考虑到上篇文章上传博客之后，图片都无法显示，尝试了很多解决方案，但是都不是很舒服。所以想等这个问题处理完成之后，会多多增加图片，增加表现力。关于这个问题，现在也已经有了很好的处理方案。所以我想着后续抽时间写一个工具专栏。专门介绍下在开发或者工作中的工具利器。一起成长，一起进步。