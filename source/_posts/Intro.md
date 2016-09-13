---
title: Intro
date: 2015-12-07 10:57:51
tags:
---

## 前言

在硬盘进水丢了所有博客的文章之后我又回来了！想想似乎从Jkyll到Octopress再到Hexo已经搭了四次环境了，每次忘记还得再去网上查，这次干脆就把流程也记一下😀

## 介绍
GitHub pages是github上用于对开源项目进行说明的网页。我们也可以根据GitHub Pages来生成个人的GitHub个人博客网站。通过GitHub Pages创建的博客属于静态博客网站，其中的内容都是HTML静态页面。(这是我从坐我隔壁的人的博客上复制粘贴的)

## 步骤

* 搭建环境
* 安装hexo
* 绑定Git库
* 相关配置
* 常用操作

## 第一步 搭建环境
1.安装Git...Mac的Xcode自带了

2.安装[Node.js](https://nodejs.org/en/)，我好像翻了墙才下成功的

## 第二步 安装hexo
```shell
sudo npm install -g hexo-cli

```
一定要加sudo认证，踩过坑。然后去你想去的目录，执行
```shell
hexo init

```
你的博客就建在当前那个目录下啦。

## 第三步 绑定Git库
之前你生成的博客只能在本地预览，因此你需建立与你用户名对应的仓库，仓库名必须为``your_user_name.github.io``比如我的就是``gxq93.github.io``，这将让你的hexo博客与你的GitHub pages绑定，不买域名的话以后这就是你博客的地址了。
接着我们来生成 ``Git SSH Key``
```shell
cd ~/.ssh
ls
```
如果没有这个文件夹或文件夹中没有id_rsa和id_rsa.pub这两个密钥和公钥，就执行
```shell
ssh -keygen
```
生成这对密钥，生成过程会提示你设置路径和密码，直接回车就是默认的``/.ssh``的路径以及没有密码
接着取出rsa
```shell
cat ~/.ssh/id_rsa.pub
```
输出大概是这样

![](https://github.com/gxq93/gxq93.github.io/blob/source/source/_posts/Intro/ssh_rsa.png)


然后把这坨东西复制到你github SSHkey设置上选择new SSHkey的key上，选择add，接着去验证一下
```shell
ssh -T git@github.com
```
如果显示的如Hi gxq93! You've successfully authenticated, but GitHub does not provide shell access.就表示绑定成功了，去github SSHkey设置上你也可以看到你的钥匙变成绿色的了。

![](https://github.com/gxq93/gxq93.github.io/blob/source/source/_posts/Intro/ssh_sucess.png)

接着打开hexo的_config.yml的配置文件，设置
```shell
deploy:
type: git
repo: git@github.com:gxq93/gxq93.github.io.git
branch: master
```
注意在冒号后面一定要加个空格！不然无法编译。

接着部署发布
```shell
hexo generate
hexo deploy
```
这两个也是平时发布必用的两个命令，接着你就会发现你的那个git库多了一些东西，至此搭建完成！
## 第四步 相关配置
很多人写了hexo的主题，你可以去网上搜一下，我之前用了next主题，现在改成了yilia，使用方法：
到你博客的目录下
```
git clone https://github.com/litten/hexo-theme-yilia.git themes/yilia
```
然后在_config.yml配置文件中设置``theme: yilia``发布一下就ok了。
但是其实因为作者的个人习惯，主题总是有一些不尽人意的地方，比如yilia中代码滑块问题，目录问题，可以另外修改配置。

1.去除代码滑块
到``/themes/yilia/source/css/_partial/highlight.styl``，增加一行``overflow: hidden``在如下位置
```
pre
@extend $code-block
color: code-word
overflow: hidden//添加
```
并在68行代码处将``height: 15px``注释掉，即可去除滑块

2.增加目录结构
因为hexo本身不支持目录结构，因此需要做些修改。到``/themes/yilia/layout/_partial/article.ejs``文件下在``<%- post.content %>``上面添加如下代码
```shell
<% if(post.toc == true){ %>
<div id="toc" class="toc-article">
<strong class="toc-title">文章目录</strong>
<%- toc(post.content, {list_number: false}) %>
</div>
<% } %>
```
修改后在文章开头设置``toc: true``，如
```shell
---
title: Intro
date: 2016-03-13 10:57:51
tags:
toc: true
---
```
即可，无需像正常md格式添加``[TOC]``

3.设置上传本地图片
对于那些想要更有规律地提供图片和其他资源以及想要将他们的资源分布在各个文章上的人来说，hexo提供了更组织化的方式来管理资源。这个稍微有些复杂但是管理资源非常方便的功能可以通过将config.yml文件中的``post_asset_folder``选项设为true，当资源文件管理功能打开后，hexo将会在你每一次通过``hexo new [layout] <title>``命令创建新文章时自动创建一个文件夹。这个资源文件夹将会有与这个markdown文件一样的名字。

使用图片标签
```
{% asset_img example.png %}
```
4.将文章存放在github上
因为hexo只是将你文章的html推送到你相应的github库上，所以如果你想要换一台电脑编写你的博客或者像我这样电脑突然坏了的话，你是无法获取到你那些文章的markdown的，而如果每次写完你都将文章放到其他地方存取也是不太方便的，而正好hexo可以选择你那个库的分支来进行推送操作，所以我采用的方式就是在那个库上新建一个分支，写完文章将``source``文件夹推送更新到那个分支上，这样你的``source``文件和相应那些博客文件都放在同一个git库上管理，更加的方便，也不会出现像我之前那样因为电脑坏了所有的文章的``source``都没有的情况了😑

## 第五步 常用操作
新建文章
```shell
hexo new "your new article"
```
部署
```shell
hexo generate
```
发布
```shell
hexo deploy
```
清除缓存
```shell
hexo clean
```
不显示全文
```shell
<!--more-->
```

最后贴出hexo及yilia网址
[https://hexo.io/](https://hexo.io/)
[https://github.com/litten/hexo-theme-yilia](https://github.com/litten/hexo-theme-yilia)
