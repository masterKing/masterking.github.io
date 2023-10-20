---
title: 揭开神秘的iOS布局
date: 2018-03-26 12:01:28
tags: iOS
---

翻译自: [Demystifying iOS Layout](http://tech.gc.com/demystifying-ios-layout/)

在你刚开始开发iOS应用时,最难避免或者说最难调试的是处理视图的布局和内容;通常这些事情的发生是因为对 **视图更新** 真实发生存在误解;了解 **视图更新** 的方式和时间需要更深入地了解iOS应用程序的主运行循环,以及它如何与`UIView`提供的某些方法关联;这篇博文将解释这些互动,希望澄清如何使用`UIView`的方法来获得你想要的行为;

## iOS应用程序的主运行循环
iOS应用程序的主运行循环用来处理所有用户输入事件并在您的应用程序中触发适当的响应;任何与应用程序的用户交互都会被添加到事件队列中;应用程序对象(如下图所示)将事件队列中的事件取出,并将它们分派给应用程序中的其他对象;**它本质上是通过解释来自用户的输入事件并在应用程序的核心对象中为该输入调用相应的处理程序来执行运行循环**;这些处理程序调用应用程序开发人员编写的代码;一旦这些方法调用返回,控制回到主运行循环并且开始更新周期;更新周期负责布局和重绘视图(在下一节中介绍);下面是应用程序如何与设备进行通信并处理用户输入的插图

![Main Event Loop](http://tech.gc.com/images/demystifying-ios-layout/main_event_loop.jpg)

*https://developer.apple.com/library/content/documentation/General/Conceptual/Devpedia-CocoaApp/MainEventLoop.html*

## 更新周期(Update Cycle)
更新周期是应用程序完成运行所有事件处理代码后,控制权返回到主运行循环的点;就是在这个点,系统开始更新布局,显示和约束;如果您要求改变视图而它正在执行事件处理程序,系统会将此视图标记为需要重绘;在下一次更新周期,系统将执行这些视图上所有的变化;用户交互和布局更新之间的时间间隔对用户来说应该是感觉不到的;iOS应用程序通常以60fps动画,这意味着一个刷新周期只需要1/60秒;由于这种情况发生的速度很快,用户不会注意到她与设备上的应用程序进行交互和看到内容和布局更新之间的UI的滞后;但是,由于事件被执行的时间和相应视图的重绘时间之间存在时间间隔,所以在运行循环过程中,视图可能不会按照您希望的方式更新;如果您有任何计算依赖视图最新的内容或布局,您有可能操作的是该视图的旧的内容或布局而不是最新的;了解运行循环,更新周期和某些的`UIView`方法能够帮助避免或者调试这类问题;

您可以在下图中看到更新周期在运行循环结束时如何发生
![Update Cycle](http://tech.gc.com/images/demystifying-ios-layout/tech-blog-loop.png)

## 布局
视图的布局指的是它在屏幕上的大小和位置;每一个视图都有一个`frame`属性来描述它在父视图坐标系统中的位置以及它的大小;`UIView`提供了一些方法,可以让你通知系统一个视图的布局已经改变,同时为你提供了可以重写的方法,以便在重新计算视图的布局后定义要执行的操作

### layoutSubviews()

这个`UIView`方法调整视图及其所有子视图的大小和位置;它给出当前视图和每个子视图的位置和大小;此方法很昂贵,因为它对视图的所有子视图起作用并调用其相应的`layoutSubviews`方法;系统会在任何需要重新计算视图的 frame 属性时调用此方法,因此当您想要设置 frame 属性指定视图位置和大小的时候您应该重写此方法;但是,当您的视图层次结构需要布局刷新时,您绝不应该显示的调用它;相反,在运行循环期间,您可以使用多种机制在不同点触发`layoutSubviews`调用,这比直接调用`layoutSubviews`方法要便宜的多;

当`layoutSubviews`方法完成时,将在拥有该视图的视图控制器中触发对`viewDidLayoutSubviews`的调用;**由于`layoutSubviews`是更新视图布局后可靠调用的唯一方法,因此应该将任何取决于布局和大小的逻辑代码方法`viewDidLayoutSubviews`中,而不是放在`viewDidLoad`或`viewDidAppear`中**;这是避免使用过时的布局或者位置变量的唯一方法。

## 自动刷新触发器
有多个事件会自动地将视图标记为布局已经改变,所以该视图的`layoutSubviews`方法将在下一次更新周期时被系统调用,不需要开发人员手动执行这个方法;

这些自动将视图标记为布局已经改变的方式有以下几种:

* 改变视图的大小
* 添加子视图
* 用户滚动`UIScrollView`(`layoutSubviews`方法会被`UIScrollView`以及它的父视图调用)
* 用户旋转设备
* 更新视图的约束

以上这些方式都告诉系统,视图的位置需要重新计算并且会自动导致最终的`layoutSubviews`方法的调用;当然,也有直接触发`layoutSubviews`方法调用的办法;

### setNeedsLayout()
触发`layoutSubviews`调用最省资源的方式就是在您的视图上调用`setNeedsLayout`方法;这将指示系统这个视图的布局需要重新计算;`setNeedsLayout`执行并立即返回,并且在返回之前并不实际更新视图;相反,视图将会在下一个更新周期(系统调用这些视图以及后续所有子视图的`layoutSubviews`方法)实际更新视图的布局;即使从`setNeedsLayout`返回后到视图被重新绘制布局之间有一段任意的时间间隔,但是这个延迟不会对用户造成影响,因为永远不会长到对界面造成卡顿;

### layoutIfNeeded()
`layoutIfNeeded`是`UIView`的另一个将会在不久后触发`layoutSubviews`调用的方法;与`setNeedsLayout`会让视图在下一个周期调用`layoutSubviews`更新视图不同,`layoutIfNeeded`会立即触发`layoutSubviews`方法调用,如果视图需要布局更新的话;如果你在调用`setNeedsLayout`方法或者触发上面描述的自动刷新触发器之后调用了`layoutIfNeeded`方法,`layoutSubviews`将会在视图上被调用;然而,如果你调用`layoutIfNeeded`之后没有动作指示系统视图需要重新刷新视图,那么`layoutSubviews`方法将不会被调用;
在一次运行循环中,两次调用视图的`layoutIfNeeded`方法之间,视图的布局并没有变化的话,那么第二次调用将不会触发`layoutSubviews`的调用;

与`setNeedsLayout`方法不同,使用`layoutIfNeeded`方法,布局和重绘会在函数返回之前立即发生改变(除非有正在运行中的动画);这个方法在你需要依赖新的布局而又无法等待视图的下次更新周期到来的时候特别有用;然而,除了这种情况外,你还是应该调用`setNeedsLayout`然后等待下次更新周期的到来,这样在每次运行循环中都只会更新一次布局;

动画更改约束时,此方法特别有用;您应该在动画的 block 开始之前调用一次`layoutIfNeeded`,以确保在动画开始之前通知所有的布局更新;配置新的约束,然后在动画 block 内,再次调用`layoutIfNeeded`以动画到最新的状态;

## 显示
视图的显示包含颜色,文本,图片和Core Graphics绘制等视图属性,但不包含它和它的子视图的大小和位置;和布局的方法类似,显示也有触发更新的方法,它们由系统在检测到更新时被自动调用,或者我们可以手动调用直接触发更新;

### draw(_:)
`UIView`的`draw`方法(Objective-C中的`drawRect`)对视图的显示内容的作用	就像 `layoutSubviews`方法对视图的位置和尺寸的作用;同`layoutSubviews`一样,你不应该在代码中直接调用`draw`方法,而应该在运行循环的不同点调用能触发`draw`方法调用的方法;然而,与`layoutSubviews`方法不同的是,`draw`方法不会触发后续子视图的调用;

### setNeedsDisplay()
这个方法类似布局中的`setNeedsLayout`;它会给有显示内容更新的视图设置一个内部的标记之后返回,并不会真正的视图重绘;而是在接下来的更新周期中,系统会遍历所有已被标记的视图,调用它们的`draw`方法;如果你只想在下次更新时重绘部分的视图,你可以调用`setNeedsDisplay(_:)`(Objective-C中的`setNeedsDisplayInRect:`)方法,并把希望重绘的矩形部分传入参数;

大部分时候,在视图中更新任何 UI 组件都会通过自动设置内部的"显示内容更新"标记将视图标记为"dirty"的;导致在下一次更新周期中视图的内容就会重绘而不需要直接显示调用`setNeedsDisplay`;然而如果你有一个属性没有绑定到UI控件,但需要在属性值每次更新重绘视图,那么你可以实现该属性的`didSet`方法,并在里面调用`setNeedsDisplay`方法来触发视图的更新;

有时设置一个属性要求自定义绘制,这种情况下你需要重写`draw`方法;在下面的例子中,设置`numberOfPoints`会触发系统根据具体点数绘制不同的视图;在这个例子中,你需要在`draw`方法中实现自定义绘制,并在`numberOfPoints`的property observer里调用`setNeedsDisplay`;

```
Class MyView: UIView {
	var numberOfPoints = 0 {
		didSet {
			setNeedsDisplay()
		}
	}
	override func draw(_ rect: CGRect) {
		switch numberOfPoints {
		case 0: 
			return
		case 1: 
			drawPoint(rect)
		case 2:
			drawLine(rect)
		case 3: 
			drawTriangle(rect)
		case 4:
			drawRectangle(rect)
		case 5: 
			drawPentagon(rect)
		default:
			drawEllipse(rect)
		}
	}
}
```
视图的显示方法里没有类似布局中的`layoutIfNeeded`这样可以触发立即更新的方法;通常情况下等到下一个更新周期再重新绘制视图也无所谓;

## 约束
在自动布局技术中布局和重绘视图有三个步骤;第一步是更新约束,系统计算并设置视图上所有必需的约束条件;第二步是布局阶段,布局引擎计算视图和子视图的 frame 并且将它们布局;最后一步完成这一循环的是显示阶段;第三步完成此次循环的是显示阶段,如果有必要,那么通过调用视图的`draw`方法重绘视图的内容;

### updateConstraints()
这个方法用在自动布局中动态的改变视图的约束;和布局中的`layoutSubviews`方法和显示内容中的`draw`方法类似,`updateConstraints`只应该被重写,而不应该在你的代码中直接调用;一般来说,您应该在`updateConstraints`方法中仅仅实现必须要更新的约束;静态的约束应该设置在interface builder,视图的初始化方法(initializer)或者控制器的`viewDidLoad`中;

通常情况下,开启或者关闭约束,更改约束的优先级或者常量值,或者从视图层级中移除一个视图时都会设置一个内部的标记,这个标记将会在下一次更新周期触发`updateConstraints`方法调用;当然啦,也有手动的给视图打上需要更新约束的标记的方法,如下:

### setNeedsUpdateConstraints()
调用`setNeedsUpdateConstraints`会保证在下一次的更新周期中更新约束;它通过标记视图的约束已更新来触发`updateConstraints`调用;这个方法和`setNeedsDisplay`,`setNeedsLayout`方法的工作机制类似;

### updateConstraintsIfNeeded()
这个方法就等同于使用了自动布局的视图中的`layoutIfNeeded`方法;它会检查视图约束是否更新的标记(能够被自动设置,或者通过`setNeedsUpdateConstraints`设置,或者通过`invalidateInstrinsicContentSize`设置),如果它表明约束需要更新,它将立刻触发`updateConstraints`方法的调用而不需要等到运行循环的结束;

### invalidateIntrinsicContentSize()
一些使用自动布局的视图中会有一个`intrinsicContentSize`的属性,这是视图根据它的内容得到的自然尺寸;一个视图的`intrinsicContentSize`属性通常由所包含的元素的约束来决定,但是也可以通过重写来提供自定义的行为;调用`invalidateIntrinsicContentSize`会设置一个标记表示这个视图的`intrinsicContentSize`已经过期,需要在下一个布局阶段重新计算;

## 它们是如何连接的
视图的布局,显示以及约束都遵循着相似的模式,例如它们更新的方式以及如何在运行循环的不同点上强制更新;任意组件都有一个实际去更新的方法(`layoutSubviews`,`draw`,以及`updateConstraints`),你可以重写来手动操作视图,但是任何情况下都不要在你的代码中直接调用;这些方法仅仅在视图有标记,告诉系统视图的某些组件需要更新了,在主运行循环的后面被调用;有些操作会自动设置这个标志,也有一些方法能够让您手动的设置它;对于布局和约束的更新,如果您无法等到更新周期的到来(因为有些操作依赖最新的布局),有这么一些方法可以让你立即更新,并保证布局需要更新标记被正确标记;下面的表格列出了任意组件会怎样更新及其对应方法;

Method purposes | Layout | Display | Constraints
--------------- | ------ | ------- | -----------
Implement updates(override,don't call explicitly) | `layoutSubviews` | `draw` | `updateConstraints`
Explicitly mark view as needing update on next update cycle | `setNeedsLayout` | `setNeedsDisplay` | `setNeedsUpdateConstraints` `invalidateIntrinsicContentSize`
Update iimmediately if view is marked as 'dirty' | `layoutIfNeeded` | | `updateConstraintsIfNeeded`
Actions that implicitly cause views to be updated | `addSubview`<br> Resizing视图,通过`setFrame`改变视图的`bounds`(不只是translation)<br>用户滑动UIScrollView<br>用户旋转设备 | 改变视图的`bounds` | 激活/禁用约束<br>更改约束的值或者优先级<br>从视图层次结构中移除视图

下面的流程图总结了**更新周期**和**事件循环**之间的交互,并指出了上文提到的方法在**运行循环**期间的位置;你可以在运行循环中的任意一点直接的调用`layoutIfNeeded`或者`updateConstraintsIfNeeded`,需要记住,这开销会很大;在循环的主运行循环的后面是更新周期,如果视图被设置特定的"需要更新约束","需要更新布局"或者"需要更新显示"的标记,在这个节点会进行更新约束,更新布局以及更新显示内容;一旦这些更新结束,主运行循环会重新开始;

![Update Cycle](http://tech.gc.com/images/demystifying-ios-layout/update_cycle.png)*https://i.stack.imgur.com/i9YuN.png*


