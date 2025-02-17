---
title: 记录一个pod install失败的解决方法
date: 2018-04-16 14:56:39
tags: iOS

---

今天在网络看到一个 demo,想把它 clone 下来运行一下看看效果,大家都知道一般 clone 下来的项目需要使用 `pod install` 命令安装一下第三方库的,这个 demo 也不例外;

问题在于这个 demo 的 cocoapods 版本太低了(0.39.0)以至于 Podfile 中有些语法现如今都无法识别...以下截图是执行 `pod install` 命令之后给出了的错误提示:

![](记录一个pod-install失败的解决方法/Snip20180416_21.png)

项目的 Podfile 如下:

``` perl
source 'https://github.com/CocoaPods/Specs.git'
use_frameworks!

target 'RTPagedCollectionViewLayout_Example', :exclusive => true do
  pod 'RTPagedCollectionViewLayout', :path => '../'
end

target 'RTPagedCollectionViewLayout_Tests', :exclusive => true do
  pod 'RTPagedCollectionViewLayout', :path => '../'
  
end
```

大概的意思就是不支持 :exclusive => true语法，在 cocoaPods 1.0 之后 exclusive 语法已经被移除了

[这里是Stack Overflow上的解答](https://github.com/CocoaPods/CocoaPods/issues/4705)

将相关的代码删除之后,注意`,`号也要删除;

``` perl
source 'https://github.com/CocoaPods/Specs.git'
use_frameworks!

target 'RTPagedCollectionViewLayout_Example' do
  pod 'RTPagedCollectionViewLayout', :path => '../'
end

target 'RTPagedCollectionViewLayout_Tests' do
  pod 'RTPagedCollectionViewLayout', :path => '../'
  
end
```

再次执行 `pod install` 命令,结果如下

![](记录一个pod-install失败的解决方法/Snip20180416_22.png)

已经安装成功了,给了两个警告⚠️,你可以不用管它,有强迫症的同学可以在 target 前加上 `platform :ios, '7.1'`,这个 7.1 是你的项目的 Deployment Target...

``` perl
source 'https://github.com/CocoaPods/Specs.git'
use_frameworks!

platform :ios, '7.1'

target 'RTPagedCollectionViewLayout_Example' do
  pod 'RTPagedCollectionViewLayout', :path => '../'
end

target 'RTPagedCollectionViewLayout_Tests' do
  pod 'RTPagedCollectionViewLayout', :path => '../'
  
end
```

再次运行 `pod install` 完美

![](记录一个pod-install失败的解决方法/Snip20180416_23.png)

