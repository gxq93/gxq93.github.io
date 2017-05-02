---
title: Swift闭包形式的初始化方法
date: 2016-10-08 14:42:26
tags: [Swift]
categories: 技术
thumbnail: http://7xtg0o.com1.z0.glb.clouddn.com/1-BRHKm8nigXNKVdF1B0PX0g.png
---
对于swift一般控件的初始化，相信大部分都还是习惯这样声明：
``` swift
lazy var customView:UIView = {
    let view:UIView = UIView(frame: CGRect(x: 0, y: 0, width: 100, height: 100))
    view.backgroundColor = UIColor.yellow
    return view
}()
```
今天发现了一个新的写法（也仅仅只是一种写法而已。。。）
``` swift
lazy var customView:UIView = {
    $0.backgroundColor = UIColor.yellow
    return $0
}(UIView(frame: CGRect(x: 0, y: 0, width: 100, height: 100)))
```
<!--more-->

最终达到的效果是一样的，当然了，实现的原理也仅仅只是调用闭包将对象返回作为变量，第一种的闭包不传参数返回自定义的对象，而第二种先将该对象进行初始化并作为闭包的参数传入进行其它的一些业务逻辑绑定。对于写无聊了第一种方法可以用用第二种来装装逼: )
或者再恶心一点，对于初始化完不需要其它操作的控件，可以这样：
``` swift
var _:UIView = {
    $0.backgroundColor = UIColor.yellow
    view.addSubview($0)
    return $0
}(UIView(frame: CGRect(x: 0, y: 0, width: 100, height: 100)))
```
当你的同事在viewDidLoad中看到这样一坨东西，是不是黑人问号，一脸懵逼这他妈是哪个东西。
当然这些只是玩笑而已，毕竟这么写可能过个几天，这坨东西你自己可能都不认识了😃
