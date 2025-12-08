---
layout: post
title: 三小时建成github pages + hexo博客
date: 2016-07-24 21:10:10
# tags: 博客
categories:
 - blog
---

因为羡慕身边大神都有个炫酷的博客，所以下午的时候花时间建立起这个博客。得益于网络教程，发现原来并不难。以下是这一过程的记录，顺便练练markdown。
<!-- more -->
## 为什么要建立这个博客
### 为什么选择GitHub Pages
-   域名是github的二级域名，不用给空间付费，不用给域名付费
-   流行又简洁的MarkDown写作语法
-   支持本地编写、本地预览
-   seo优化上，github在google上权重高

### 为什么我要写博客
-   本人表达能力不佳，把脑里的东西写下来能让我印象更深刻，提高我将事情讲清楚的能力和逻辑思维能力
-   有利于我积累更多的知识，并享受分享带来的连锁反应
-   用markdown写东西觉得自己很geek

## 准备步骤
1.  安装[node.js](https://nodejs.org/)
2.  安装[git](https://git-scm.com/)
3.  注册[github](https://www.github.com/)

## 配置SSH keys
### 生成SSH Keys
建立博客之前要先用SSH keys让我们的本地git项目与远程的github建立联系。
首先我们需要检查你电脑上现有的ssh key。右键打开Git Bash，输入：
（如果提示：No such file or directory 说明你是第一次使用git。）
``` bash
$ cd ~/. ssh
```

生成新的SSH Key
（此处的邮箱地址请输入你自己的邮箱地址）
``` bash
$ ssh-keygen -t rsa -C "邮件地址@youremail.com"
```
然后回车

然后系统会要你输入密码。这个密码会在你提交项目时使用，如果为空的话提交项目时则不用输入。
（输入密码的时候没有*字样的，你直接输入就可以了）
``` bash
Enter passphrase (empty for no passphrase):<输入加密串>
Enter same passphrase again:<再次输入加密串>
```
看到这样的界面就成功设置ssh key了。
![成功设置 ssh key](/assets/img/2016/07/1.png)

注意：如果你使用的是mac，需要在命令前加上sudo，详细可以看这篇文章[Hexo部署时提示Fatal： Could not read from remote repository的问题处理](https://idealife.github.io/2015/10/02/Hexo%E9%83%A8%E7%BD%B2%E6%97%B6%E6%8F%90%E7%A4%BAfatal-Could-not-read-from-remote-repository%E7%9A%84%E9%97%AE%E9%A2%98%E5%A4%84%E7%90%86/)

### 将SSH Key添加到Github上
1.  打开本地C:\Documents and Settings\Administrator\.ssh\id_rsa.pub文件。路径也有可能是C:\Users\Administrator\.ssh，你可以直接在C盘中查找id_rsa.pub文件。此文件里面内容为刚才生成人密钥。如果看不到这个文件，你需要设置显示隐藏文件。打开，准确的复制这个文件的内容。
2.  登陆github系统。点击右上角的图像--->Settings ---> SSH and GPG keys。
![将SSH Key添加到Github上](/assets/img/2016/07/2.png)
3.  点击右上角New SSH key，把你刚刚复制的本地生成的密钥文件内容黏贴到里面（Key文本框中）， 点击Add SSH key就ok了
![将SSH Key添加到Github上](/assets/img/2016/07/3.png)

### 测试一下
可以输入下面的命令，看看设置是否成功，git@github.com的部分不要修改：
``` bash
$ ssh -T git@github.com
```
然后输入yes
然后就会看到
``` bash
Hi seminelee! You've successfully authenticated, but GitHub does not provide shell access.
```

### 设置用户信息
现在你已经可以通过SSH链接到GitHub了，还有一些个人信息需要完善的。
Git会根据用户的名字和邮箱来记录提交。GitHub也是用这些信息来做权限的处理，输入下面的代码进行个人信息的设置，把名称和邮箱替换成你自己的。
``` bash
$ git config --global user.name "seminelee"//用户名
$ git config --global user.email  "may.air@qq.com"//填写自己的邮箱
```
完成以上步骤后本机就已成功连接到github


## 开始建立博客
与GitHub建立好链接之后，就可以方便的使用它提供的Pages服务，GitHub Pages分两种，一种是你的GitHub用户名建立的username.github.io这样的用户&组织页（站），另一种是依附项目的pages。
想建立个人博客是用的第一种，形如seminelee.github.io这样的可访问的站，每个用户名下面只能建立一个。

### Github上建立仓库
登陆Github，建立一个名为seminelee.github.io的仓库。
注意！Github Pages的Repository名字是特定的，比如我Github账号是seminelee，那么我Github Pages Repository名字就是seminelee.github.io。
详细建立仓库过程略过。

### 安装Hexo
Hexo是一个简单、快速、强大的博客发布工具，支持Markdown格式。
打开Git Bash
``` bash
$ npm install -g hexo
```
安装完毕后，在我的电脑某个位置中建立一个名字叫hexo的文件夹，然后在此文件夹中右键打开Git Bash。
``` bash
$ hexo init
```
Hexo随后会自动在目标文件夹建立网站所需要的所有文件。
现在我们已经搭建起本地的hexo博客了。
在hexo目录下输入
``` bash
$ hexo g
$ hexo s
```
然后到浏览器输入localhost:4000看看，可以看到默认主题下的博客，这就实现了本地预览了。

### 更换主题
每次更换主题前清空一下database
``` bash
$ hexo clean
$ hexo g
$ hexo s
```
通过git clone克隆主题，这里用的是next主题
``` bash
$ git clone https://github.com/iissnan/hexo-theme-next.git
```
更多主题可以参考[有哪些好看的 Hexo 主题? - GitHub - 知乎](https://www.zhihu.com/question/24422335)

启用主题
``` bash
theme: hexo-theme-next
```
修改hexo目录下的config.yml配置文件中的theme属性，将其设置为hexo-theme-next。
（注意：Hexo有两个config.yml文件，一个在根目录，一个在theme下，此时修改的是在根目录下的。）

更新主题
``` bash
$ cd themes/hexo-theme-next
$ git pull
```

然后可以本地预览一下
``` bash
$ hexo g #generate生成
$ hexo s #server测试环境，启动本地服务，进行文章预览调试
```

### 上传到Github仓库
打开hexo目录下的_config.yml，拉到最下面
配置为这样子,只需要把seminelee改为你自己的github用户名就可以了。（注意格式，冒号后要有空格，你可以直接复制以下代码再作修改）
``` bash
# Deployment
## Docs: https://hexo.io/docs/deployment.html
deploy:
  type: git
  repository: https://github.com/seminelee/seminelee.github.io.git
  branch: master
```
然后执行命令
``` bash
$ hexo g #generate生成
$ hexo d #deploy开发环境
```
如果看到结果最后一行是INFO Deploy done:git则没有问题。否则，则可以把上面的配置改为下面这种使用SSH方式的提交，把用户名改为你自己的用户名
``` bash
# Deployment
## Docs: https://hexo.io/docs/deployment.html
deploy:
  type: git
  repository: git@github.com:seminelee/seminelee.github.io.git
  branch: master
```
再次执行命令
``` bash
$ hexo g #generate生成
$ hexo d #deploy开发环境
```
如果在执行 hexo deploy 后,出现 error deployer not found:github 的错误，网上说是hexo 更新到3.0之后的一个坑，则需要安装hexo-deployer-git
``` bash
$ npm install hexo-deployer-git --save
```
再次执行命令之后，打开[https://seminelee.github.io/](https://seminelee.github.io/)就能看到你建立好的博客了！

## 参考
-   [如何搭建一个独立博客——简明Github Pages与Hexo教程](https://www.jianshu.com/p/05289a4bc8b2)
-   [使用github+Hexo人人都能拥有一个美美的博客](https://www.jianshu.com/p/863f3f2d1733)
-   [搭建 hexo，在执行 hexo deploy 后,出现 error deployer not found:github 的错误](https://www.v2ex.com/t/175940)
