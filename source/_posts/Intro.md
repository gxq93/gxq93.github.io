---
title: 基于Hexo的个人博客
date: 2015-6-07 10:57:51
tags: [Hexo]
categories: 技术
---

## 前言

在硬盘进水丢了所有博客的文章之后终于又折腾回来了！想想似乎从 Jkyll 到 Octopress 再到 Hexo 已经搭了三次环境了，每次步骤还得再去网上查，这次干脆就把流程也记一下，防止以后忘记了。

<!--more-->

## 介绍

GitHub pages 是 github 上用于对开源项目进行说明的网页。我们也可以根据 GitHub Pages 来生成个人的 GitHub 个人博客网站。通过 GitHub Pages 创建的博客属于静态博客网站，其中的内容都是 HTML 静态页面。

## 步骤

* 搭建环境
* 安装hexo
* 绑定Git库
* 相关配置
* 常用操作

## 第一步 搭建环境

1.安装 Git，Mac 的 Xcode 自带了

2.安装[Node.js](https://nodejs.org/en/)

## 第二步 安装hexo

```
sudo npm install -g hexo-cli
```
一定要加 sudo 认证，踩过坑。然后去你想去的目录，执行
```
hexo init
```
你的博客就建在当前那个目录下了。

## 第三步 绑定Git库

之前你生成的博客只能在本地预览，因此你需建立与你用户名对应的仓库，仓库名必须为``your_user_name.github.io``比如我的就是``gxq93.github.io``，这将让你的 hexo 博客与你的 GitHub pages 绑定，不买域名的话以后这就是你博客的地址了。
接着我们来生成 ``Git SSH Key``：
```
cd ~/.ssh
ls
```
如果没有这个文件夹或文件夹中没有 id_rsa 和 id_rsa.pub 这两个密钥和公钥，就执行：
```
ssh-keygen
```
生成这对密钥，生成过程会提示你设置路径和密码，直接回车就是默认的``/.ssh``的路径以及没有密码。
接着取出 rsa 公钥：
```
cat ~/.ssh/id_rsa.pub
```
输出大概是这样

{% asset_img ssh_rsa.png %}

然后把这坨东西复制到你 github SSHkey 设置上选择 new SSHkey 的 key 上，选择 add，接着去验证一下
```
ssh -T git@github.com
```
如果显示的如 Hi gxq93! You've successfully authenticated, but GitHub does not provide shell access. 就表示绑定成功了，去 github SSHkey 设置上你也可以看到你的钥匙变成绿色的了。

{% asset_img ssh_sucess.png %}

接着打开 hexo 的 _config.yml 的配置文件，设置
```
deploy:
type: git
repo: git@github.com:gxq93/gxq93.github.io.git
branch: master
```
注意在冒号后面一定要加个空格！不然无法编译。

另外下载 hexo 通过 git 发布的插件
```
sudo npm install hexo-deployer-git --save
```

接着部署发布
```
hexo generate
hexo deploy
```
这两个也是平时发布必用的两个命令，接着你就会发现你的那个 git 库多了一些东西，至此搭建完成！

Tips：设置上传本地图片

对于那些想要更有规律地提供图片和其他资源以及想要将他们的资源分布在各个文章上的人来说，hexo 提供了更组织化的方式来管理资源。这个稍微有些复杂但是管理资源非常方便的功能可以通过将``config.yml``文件中的``post_asset_folder``选项设为 true，当资源文件管理功能打开后，hexo 将会在你每一次通过``hexo new [layout] <title>``命令创建新文章时自动创建一个文件夹。这个资源文件夹将会有与这个 markdown 文件一样的名字。

使用图片标签

```
{% asset_img example.png %}
```

Tips：将文章存放在 github 上

因为 hexo 只是将你文章的 html 推送到你相应的 github 库上，所以如果你想要换一台电脑编写你的博客或者像我这样电脑突然坏了的话，你是无法获取到你那些文章的 markdown 的，而如果每次写完你都将文章放到其他地方存取也是不太方便的，而正好 hexo 可以选择你那个库的分支来进行推送操作，所以我采用的方式就是在那个库上新建一个分支，写完文章将``source``文件夹推送更新到那个分支上，这样你的 source 文件和相应那些博客文件都放在同一个 git 库上管理，更加的方便，也不会出现像我之前那样因为电脑坏了所有的文章的 source 都没有的情况了。

## 第四步 常用操作
新建文章
```
hexo new "your new article"
```
部署
```
hexo generate
```
发布
```
hexo deploy
```
清除缓存
```
hexo clean
```
不显示全文
```
<!--more-->
```

最后贴出hexo网址
[https://hexo.io/](https://hexo.io/)
