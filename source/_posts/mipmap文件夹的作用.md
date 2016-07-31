---
title: mipmap文件夹的作用
date: 2016-02-04 14:33:24
tags: [Android,Note]
---

简述res/目录下的drawable/目录和mipmap/目录的区别

<!--more-->

---

在Android Studio中新建一个Project的时候，相比Eclipse创建时res/目录内会多几个mipmap文件夹

![mipmap](https://raw.githubusercontent.com/bluehan/Images/master/Other/mipmap.png)

刚开始以为是mipmap是取代drawable来存放图片文件的，后来看到官网的这篇文章才知道mipmap文件夹只是Android推荐用来存放app的launcher icon的目录。

#### **原文地址:**
+ [Managing Launcher Icons as mipmap Resources](http://developer.android.com/intl/zh-cn/tools/projects/index.html#mipmap)

> 原文:
> Different home screen launcher apps on different devices show app launcher icons at various resolutions. When app resource optimization techniques remove resources for unused screen densities, launcher icons can wind up looking fuzzy because the launcher app has to upscale a lower-resolution icon for display. To avoid these display issues, apps should use the mipmap/ resource folders for launcher icons. The Android system preserves these resources regardless of density stripping, and ensures that launcher apps can pick icons with the best resolution for display.Make sure launcher apps show a high-resolution icon for your app by moving all densities of your launcher icons to density-specific res/mipmap/ folders (for example res/mipmap-mdpi/ and res/mipmap-xxxhdpi/). The mipmap/ folders replace the drawable/ folders for launcher icons. For xxhpdi launcher icons, be sure to add the higher resolution xxxhdpi versions of the icons to enhance the visual experience of the icons on higher resolution devices.
> > **Note: Even if you build a single APK for all devices, it is still best practice to move your launcher icons to the mipmap/ folders.**

最后一段也提到了，最佳的实践方法就是讲你的launcher icons全部移动到mipmap/文件夹下

从文章的开头可以看到，其实在Android Studio中创建Project的时候，默认已经将所有的launcher icons移动到mipmap/文件夹下了。所以存放其他图片还得存放在drawable下。

##### **另外附上drawable/文件夹官方的描述：**
> For bitmap files (PNG, JPEG, or GIF), 9-Patch image files, and XML files that describe Drawable shapes or Drawable objects that contain multiple states (normal, pressed, or focused). See the Drawable resource type.
