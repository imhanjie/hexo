title: 「Android-Art」理解Drawable
date: 2016-07-25 16:01:44
tags: [Android,Notes,Android开发艺术探索]
---

Drawable在开发中有着自己的有点：首先，它使用简单，比定义View的成本要低；其次，非图片类型的Drawable占用空间较小，这对减小APK的大小也很有帮助

<!--more-->

### 以下整理自「Android开发艺术探索」第六章 ###

---


### 一、Drawable简介 ###

Drawable有很多种，他们都表示一种图像概念，但是他们又不全是图片，通过颜色也可以构造出各式各样的图像效果。在实际开发中，Drawable常被用来作为View的背景使用。Drawable一般都是通过XML来定义的，当然我们也可以通过代码来创建具体的Drawable对象，只是用代码创建会稍显复杂

Drawable的内部宽/高这个参数比较重要，通过getIntrinsicWidth和getIntrinsicHeight这两个方法可以获取到他们。但是并不是所有的Drawable都有内部宽/高，比如一张图片所形成的Drawable，它的内部宽/高就是图片的宽/高，但是一个颜色所形成的Drawable，他就没有内部宽/高的概念。另外需要注意的是，Drawable的内部宽/高不等同于它的大小，一般来说，Drawable时没有大小概念的，当用作View的背景时，Drawable会被拉伸至View的同等大小。

### 二、Drawable的常见分类 ###

#### 1、BitmapDrawable ####
#### 2、ShapeDrawable ####
#### 3、LayerDrawable ####
#### 4、StateListDrawable ####
#### 5、LevelListDrawable ####
#### 6、TransitionDrawable ####
#### 7、InsetDrawable ####
#### 8、ScaleDrawable ####
#### 9、ClipDrawable ####

> 具体使用见「Android开发艺术探索」第六章
