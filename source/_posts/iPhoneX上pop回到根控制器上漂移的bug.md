---
title: iPhoneX上pop回到根控制器上漂移的bug
date: 2018-04-28 16:02:08
tags: iOS
---

记录一个在iPhone X上发生的诡异的bug...语言怎么描述都太苍白,那么直接看图
![](https://raw.githubusercontent.com/masterKing/markDownPictures/master/BrowserPreview_tmp201804281808.gif)

只有在滑动到最底部的时候,push到下一个页面,然后在pop回来就会出现contentOffset.y值自动偏移的现象...

视图的层次结构如图:
![](https://raw.githubusercontent.com/masterKing/markDownPictures/master/Snip20180428_1.png)

选中的视图控制器就是TabBarController的第二个子控制器,控制器的view就是一个UICollectionView;我是很懵逼的...同事说可能是iPhone X上的安全距离的原因(但我还是很懵逼)...于是我对视图层次结构做了下修改;

* 取消修改控制器的view为UICollectionView
* 将UICollectionView作为控制器的view的子视图
* 设置collectionView的约束为,上左右等于控制器的view,下等于控制器的view的下面,但是偏移一个-34的高度(仅在iPhone X上)

修改之后的视图层次结构如下:
![](https://raw.githubusercontent.com/masterKing/markDownPictures/master/Snip20180428_2.png)

这样,collectionView不再漂移了...对上述偏移的值进行修改测试可以发现,当这个值小于等于-34的时候就不会发生漂移,大于-34时就会发生漂移...换一句话的意思就是,如果collectionView距离底部的距离小于34的那么就会漂移,大于等于34不会发生漂移...