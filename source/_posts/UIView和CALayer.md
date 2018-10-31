---
title: UIView和CALayer
date: 2017-09-24 18:05:23
tags:
---

# UIView和CALayer
> 在实际的开发经常使用`UIView`以及其子类和`CALayer`,或者设置`CALayer`的属性达到某些视觉效果。所以会有这样的疑惑：为什么要单独出一个`CALayer`呢？`CALayer`和`UIView`有什么区别呢？

要想解决上述疑惑，我们先来讨论他们之间的联系和区别。

### UIView和CALayer的联系
**UIView：**是一个屏幕上显示的矩形框，他们之间可以相互嵌套，呈现出树形结构。一个`UIView`可以管理它的所有子视图的位置。

**CALayer：**同样也包含一些内容（文本，图片等），也是被层级关系树管理的矩形块。

`UIView`之所以能够显示其中的内容，其实这都是`CALayer`的功劳，准确的说是`CALayer`的`contents`属性。每一个`UIView`都有一个`CALayer`实例的图层属性，也就是所谓的`backing layer`，`UIView`的职责就是创建并管理这个图层，以确保当子视图在层级关系中添加或者被移除的时候，他们关联的图层也同样对应在层级关系树当中有相同的操作就好比下图：
![](UIView和CALayer/layer.png)
另外，`UIView`作为`CALayer`的`delegate`，实现了`CALayerDelegate`，所以`UIView`能够在`CALayer`需要更新内容的时候做出相应动作。

### UIView和CALayer的区别
`UIView`和`CALayer`的最大区别在于***是否能够响应用户的操作***，所有和用户操作都是由`UIView`响应。改变`CALayer`的`animable`属性的时候，会伴随隐性动画。但是改变`UIView`的属性以达到相同的视觉效果的时候，是没有隐性动画的，原因在于`UIView`禁用了`CALayer`的隐性动画。

### 为什么不把UIView和CALayer合并为一个类？
原因在于要做职责分离，实现代码在不同平台之间的共用。在`iOS`平台的多点触控的用户界面和在`MacOS`平台的基于鼠标键盘有着本质的区别。这就是为什么`iOS`有`UIKit`和`UIView`，但是`MacOS`有`AppKit`和`NSView`的原因。他们功能上很相似，但是在实现上有着显著的区别。

但是类似于绘图，布局和动画这样的通用功能，两个平台上都一样，可以抽象出来做成独立的`Core Animation`框架， 苹果就能够在iOS和MacOS之间共享代码，使得对苹果自己的OS开发团队和第三方开发者去开发两个平台的应用更加便捷。

实际上，这里并不是两个层级关系，而是四个，每一个都扮演不同的角色，除了视图层级和图层树之外，还存在呈现树和渲染树。

### 何时使用UIView何时使用CALayer?
其实了解了他们的区别，在什么情况下使用就很清楚。一般情况下也不会有什么问题。但是当满足以下条件的时候，你可能更需要使用CALayer而不是UIView：
* 开发同时可以在Mac OS上运行的跨平台应用
* 使用多种CALayer的子类（如果对CALayer的子类感兴趣可以看看[这篇文章](https://www.raywenderlich.com/402-calayer-tutorial-for-ios-getting-started)），并且不想创建额外的 UIView去包封装它们所有。
* 做一些对性能特别挑剔的工作，比如对UIView一些可忽略不计的操作都会引起显著的不同。比如一些对性能要求的功能使用CALayer要比UIView好些。可以看看[这篇文章](https://blog.ibireme.com/2015/11/12/smooth_user_interfaces_for_ios/)。

### 参考资料
* 《iOS核心动画高级技巧》
* [View-Layer 协作](https://objccn.io/issue-12-4/)