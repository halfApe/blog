---
title: 本机多git账户管理
---
### 问题描述：

​	Idea更新代码执行pull操作时一直提示输入密码！

![](https://raw.githubusercontent.com/Alvin33/images/master/git-git_password_tip.png)

### 问题分析：

上周五的时候一直在捣鼓自己的[博客](<https://alvin33.gitee.io/>),用的是自己的github账号保存博客源代码，当时修改了本地的git秘钥，所以导致这个问题的出现。

### 问题解决：

基于上述的问题，我们只要在本机配置两个git秘钥就可以了 ，那么怎么配置的呢。网上也有很多教程，比较简单的一种做法如下：

​	分别执行以下命令：

```bash
#1.进入到本机ssh目录
cd ~/.ssh
#2.生成对应的git账户的秘钥
ssh-keygen -t rsa -C "yourgithub账户"
#此时会弹出以下内容
`Generating public/private rsa key pair.`
`Enter file in which to save the key (/c/Users/watt/.ssh/id_rsa):`
#3.在上述冒号后面修改文件名例如id_rsa_github
#4.点击回车之后会有两次输入密码操作，我们选择不输入，直接敲击回车。（输入密码以后之后操作要频繁输入密码）
#5.完了之后执行ls命令，我们会发现在该目录下生成了两个文件
`id_rsa_github id_rsa_github.pub` 

#6.针对gitlab账户，重复上述操作。（记得第三步要修改不同的文件名哦~,还有别的git账户的话，重复该操作）
#操作玩上述的步骤之后，我们会发现，当前目录下多了两个文件
`id_rsa_github id_rsa_github.pub id_rsa_gitlab id_rsa_gitlab.pub` 

#7.有了这些秘钥之后，我们将相应的is_rsa_XXX.pub中的信息配置到我们的相应的网站中。
#8.也是最后一步，我们要在本机新建一个config文件，里面的内容如下
#github
Host github.com
HostName github.com
User yourgithub账户
IdentityFile ~/.ssh/id_rsa_github

#gitlab
Host gitlab.com
HostName gitlab.com
User yourgitlab账户
IdentityFile ~/.ssh/id_rsa_gitlab
```

### 总结：

在单机多git账号的情况下，我们就可以按照上述操作配置啦,无论有多少个账户，都不用怕了。

### 最后：

生怕你不知道github或者是gitlab网站中ssh公钥在哪里配置，贴图镇楼。

![](https://raw.githubusercontent.com/Alvin33/images/master/git-ssh_github.png)

gitlab网站ssh配置在哪里？如果你找不到的话，那就点击左侧联系方式，加我好友，我告诉你~



