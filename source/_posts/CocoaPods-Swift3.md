---
title: CocoaPods Trunk上传Swift3.0项目
date: 2016-09-30 10:42:10
tags: [Swift,CocoaPods]
categories: 技术
---

起因是因为无聊，突然想提高一下我那可怜的 github 的提交数，所以随便拿了个 Swift3.0 写的轮播图组件打算传到 Cocoapods 上。写完 podspec(这个项目叫 GYBanner 🙈)然后由于才刚升级 Swift3.0 的原因，Cocoapods 也没有做好很贴心的适配，于是踩了一些坑～

<!--more-->

上传Trunk时：
```
pod trunk push GYBanner.podspec
```
报错了，这啥情况，检查一下原因

```
pod lib lint GYBanner.podspec --verbose
```
报错原因:
> Use Legacy Swift Language Version” (SWIFT_VERSION) is required to be configured correctly for targets which use Swift. Use the [Edit > Convert > To Current Swift Syntax…] menu to choose a Swift version or use the Build Settings editor to configure the build setting directly.

这什么鬼，可能是因为xcode8升级之后swift版本变成可选的，所以检查运行的时候不知道跑那个版本的吧，于是去看了一下果然cocoapods前不久更新了版本：
> 1.1.0.rc.2 (2016-09-13)
> Enhancements
> Use the SWIFT_VERSION when linting pods. To lint with Swift 3.0 add a Swift Version file. echo "3.0" >> .swift-version.

作者也说明了：
> Will default to SWIFT_VERSION = 2.3, but will check for a .swift-version file and use the version specified there if present.

所以现在 swift 版本默认跑的是2.3的，要想检查 swift3.0 的项目，需创建一个 .swift-version 的文件，在文件中输入3.0。好吧，虽然有点奇怪，但项目就是 swift3.0 写的，所以还是照着做吧，先升级一下 cocoapods
```
sudo gem install cocoapods --pre
pod --version
1.1.0.beta.2
```
一看，奇了怪了为啥更新后的 cocoapods 是 1.1.0.beta.2 版的，最新的不应该是 1.1.0.rc.2 吗，于是在 cocoapods issue 中找啊找终于发现原来是 Chinese 的痛，很多我大天朝的人想必都把 ruby 源替换成了淘宝的镜像，而我之前也不知道啥时候换成了阿里云的镜像。。。居然因为这个原因更新不到最新版。后来找到一个不错的镜像``https://gems.ruby-china.org``替换了一下，终于上传成功了。
写这篇文章也纯属无聊，可能之后 cocoapods 也会把 swift 版本的控制配置在 podspec 中吧，这样这篇文章也没有什么意义了，不过 who care: )
