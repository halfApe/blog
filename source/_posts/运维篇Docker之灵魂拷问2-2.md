---
title: 运维篇Docker之灵魂拷问2-1
---

上节中，我们已经对Docker核心组件image进行了剖析，这节我们将对image层之上的container进行拷问。从下图中，我们可以看出container是基于Image层产生的。而image与container的关系 ，我们可以类比类和对象的关系进行理解。在docker的世界中，image是只读的，而container是可读写的。

![](https://raw.githubusercontent.com/Alvin33/images/master/docker-container.png)

docker初见本尊篇我们已经介绍了如何创建一个容器，例如创建tomcat命令如下

```shell
docker run -d --name my-tomcat -p 9090:8080 tomcat
```

## 从container反推image

> 灵魂拷问1:既然我们知道container是基于image创建的，那么我们是否可以通过container反推得到image呢？

下面我们看看个例子实践一波

```shell
#1.拉取centos镜像
docker pull centos 
#2.根据centos镜像创建出centos container
docker run -d -it --name my-centos centos
#3.进入my-centos容器中
docker exec -it my-centos bash 
#4.输入vim命令，显示
bash: vim: command not found
#5.我们要做的是 对该container进行修改，也就是安装一下vim命令，然后将其生成一个新的centos #6.在centos的container中安装vim 
yum install -y vim
#7.退出容器（exit或者Ctrl+d），将其生成一个新的centos，名称为"vim-centos-image"
docker commit my-centos vim-centos-image
#8.查看镜像列表，并且基于"vim-centos-image"创建新的容器
docker run -d -it --name my-vim-centos vim-centos-image
#9.进入到my-vim-centos容器中
docker exec -it my-vim-centos bash
#10.检查vim命令是否存在
vim
```

经过以上操作之后，我们发现，我们通过container反向创建的image，再将其生成新的container就继承了原来容器的特性。不过这种方式，并不推荐使用。因为我们常规的操作是利用Docker file生成image，这样做的话，我们无法得知image的创建过程 。更加推荐的方式是上节内容中的自定义image

## container常用命令

上面我们用了几个操作容器的命令，这里做一个总结：

```shell
#1.根据镜像创建容器
docker run -d --name -p 9090:8080 my-tomcat tomcat
#2.查看运行中的container
docker ps
#3.查看所有的container[包含退出的]
docker ps -a 
#4.删除container
docker rm containerid 
#5.删除所有container
docker rm -f $(docker ps -a)
#6.进入到一个container中
docker exec -it container bash
#7.根据container生成image
docker commit my-centos vim-centos-image
#8.查看某个container的日志
docker logs container
#9.查看容器资源使用情况
docker stats
#10.查看容器详情信息
docker inspect container
#11.停止/启动容器
docker stop/start container
```

### 补充：删除image和container命令

```shell
#1.杀死所有正在运行的容器
docker kill $(docker ps -a -q)
#2.删除所有已经停止的容器
docker rm $(docker ps -a -q)
#3.删除所有未打 dangling 标签的镜
docker rmi $(docker images -q -f dangling=true)
#4.删除所有镜像
docker rmi $(docker images -q)
#5.强制删除 无法删除的镜像
docker rmi -f <IMAGE_ID>
docker rmi -f $(docker images -q)
```

```shell
 ~/.bash_aliases
	#1.杀死所有正在运行的容器.
	alias dockerkill='docker kill $(docker ps -a -q)'
 	#2.删除所有已经停止的容器.
	alias dockercleanc='docker rm $(docker ps -a -q)'
	#3.删除所有未打标签的镜像.
	alias dockercleani='docker rmi $(docker images -q -f dangling=true)'
 	#4.删除所有已经停止的容器和未打标签的镜像.
	alias dockerclean='dockercleanc || true && dockercleani'
```

## 

## Container的资源监控

顾名思义，作为容器，他必然是有自己的“大小”，只有合理的利用好容器的资源，才能让他更好地发挥作用。

上面，我们了解了Container的基本操作命令，可以执行docker stats来具体看一下docker容器的状态信息

![](https://raw.githubusercontent.com/Alvin33/images/master/Container-resource.png)

我们可以看到，默认内存分配使用了宿主机的最大内存，这样的话会让我们的资源不可控。

对于资源的限制，我们主要可以通过两方面来进行处理：

- 内存限制

  ```shell
  #--memory memory limit
  docker run -d --memory 100M --name tomcat1 tomcat
  ```

- cpu限制

  ```shell
  #--cpu-shares cpu权重
  docker run -d --memory 100M --cpu-shares 10 --name tomcat02 tomcat
  ```

![](https://raw.githubusercontent.com/Alvin33/images/master/container-sourceLimit.png)

从上图我们可以看出，已经设置成功啦。具体的大小可以根据实际情况灵活调整。

命令行我们自己玩玩看看还行，但是在生产环境，海量容器的时候，就捉襟见肘了。这时候，得请出我们强大的容器监控工具

### Weavescope

> github地址：https://github.com/weaveworks/scope

这个工具很强大，搭建也很简单，不过可能会存在一点小障碍，但也容易解决。

```shell
#1.配置weavescope环境
sudo curl -L git.io/scope -o /usr/local/bin/scope
sudo chmod a+x /usr/local/bin/scope
scope launch 192.168.110.164
#2.页面访问http://192.168.110.164:4040/，此时可能会出现页面无法访问的问题.我们要检查下4040端口是否启用。
	#2.1 查看端口状态
	firewall-cmd --query-port=4040/tcp
	#2.2 添加端口
	firewall-cmd --add-port=4040/tcp --permanent
	#2.3 端口重载
	firewall-cmd --reload
```

针对weave的详细介绍：

​	可以参考官网： https://www.weave.works/docs/ 

​	也可以看看这篇文章： https://www.jianshu.com/p/1155b97bfdd8 

## Container的技术支撑

上面我们说了一些container比较重要的点，那么总结下来说

> Container是一种轻量级的虚拟化技术，不用模拟硬件创建虚拟机。 Docker是基于Linux Kernel的Namespace、CGroups、UnionFileSystem等技术封装成的一种自 定义容器格式，从而提供一套虚拟运行环境。 
>
> Namespace：用来做隔离的，比如pid[进程]、net[网络]、mnt[挂载点]等 
>
> CGroups: Controller Groups用来做资源限制，比如内存和CPU等 
>
> Union file systems：用来做image和container分层 

想要进一步探究，就看我下一篇文章运维篇Docker之网络剖析吧，敬请期待~