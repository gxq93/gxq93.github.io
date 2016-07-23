---
title: 使用Runtime修改UIAlertController属性
date: 2016-04-20 13:51:32
tags:
---
这是在一次需求中实现的效果，需求目的是自定义``alertview``的文字样式，但是``alertview``的``message``属性修改不了，因此想到可以用runtime遍历``UIAlertController``的属性，拿到其对应的属性并修改，具体实现过程如下：

<!--more-->

## 1.获取UIAlertController方法名

代码：

```
unsigned int count = 0;
Method *alertMethod = class_copyMethodList([UIAlertController class], &count);
for (int i = 0; i < count; i++) {
    SEL name = method_getName(alertMethod[i]);
    NSString *methodName = [NSString stringWithCString:sel_getName(name) encoding:NSUTF8StringEncoding];
    NSLog(@"methodName ============= %@",methodName);
}
```
打印结果如下：

{% asset_img methodName.png %}
其实打印这个没卵用🙃

## 2.获取UIAlertController属性名

代码：

```
unsigned int count = 0;
Ivar *property = class_copyIvarList([UIAlertController class], &count);
for (int i = 0; i < count; i++) {
    Ivar var = property[i];
    const char *name = ivar_getName(var);
    const char *type = ivar_getTypeEncoding(var);
    NSLog(@"%s =============== %s",name,type);
}
```
打印结果如下：
{% asset_img propertyName.png %}
大致感觉带message的那两个属性应该有用，尝试把第三个``_attributedMessage``属性改掉

代码：

```
Ivar message = property[2];

/**
*  字体修改
*/
UIFont *big = [UIFont systemFontOfSize:25];
UIFont *small = [UIFont systemFontOfSize:18];
UIColor *red = [UIColor redColor];
UIColor *blue = [UIColor blueColor];
NSMutableAttributedString *str = [[NSMutableAttributedString alloc]initWithString:@"hello world" attributes:
@{NSFontAttributeName:big,NSForegroundColorAttributeName:red}];
[str setAttributes:@{NSFontAttributeName:small} range:NSMakeRange(0, 2)];
[str setAttributes:@{NSForegroundColorAttributeName:blue} range:NSMakeRange(0, 4)];

//最后把message内容替换掉
object_setIvar(_alert, message, str);
```
运行一下，bingo!
{% asset_img result.png %}

最后贴出[demo](https://github.com/gxq93/Runtime_UIAlertViewController)地址
