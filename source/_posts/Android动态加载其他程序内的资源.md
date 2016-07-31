title: Android动态加载其他程序内的资源
date: 2016-07-28 15:01:30
tags: [Android,Notes]
---


一个简单的插件化思路

<!--more-->

---

> 今天在写自己的天气app的时候想给自己app额外多提供一套天气图标可供替换。

这里有两种思路：
- 1、直接将这一套天气图标放进项目中
- 2、将一套天气图标放在一个单独的apk中，以插件的形式提供

第一种方法是很简单的，这里只要讨论第二种方法的实现过程

##### 首先访问另一个app中的资源 #####

例如访问com.melodyxxx.flymestyle包中名称为yuyu的drawable
``` java
Resources resource = null;
try {
    resource = getPackageManager().getResourcesForApplication("com.melodyxxx.flymestyle");
} catch (PackageManager.NameNotFoundException e) {
    e.printStackTrace();
}
if (resource != null) {
    int id = resource.getIdentifier("com.melodyxxx.flymestyle:drawable/yuyu", null, null);
    if (id != 0) {
        Drawable drawable = resource.getDrawable(id);
        // do sth...
    }
}
```

这里要注意的是，获取到id后，需要用另一个程序的resource来获取对应的drawable，不能直接在主app内根据此id将一个imageview设为背景，因为这样做，会直接用主app的resource根据id去主app内寻找对应的drwable，显然这是不对的，除drawable之外还可以获取string等类型的资源。

##### 另外还需要考虑一种情况，如何在RemoteViews中设置来自另一app中的资源？ #####

由于RemoteViews的特殊性，对于ImageViews设置背景，RemoteViews主要提供了2个方法：
``` java
remoteViews.setImageViewBitmap(int viewId, Bitmap bitmap)
remoteViews.setImageViewResource(int viewId, int srcId);
```
这里我们只能使用第一种方法`setImageViewBitmap`，若使用setImageViewResource，srcId我们不能直接去使用获取到的id，原因上面已经说明，所以使用第一种方法需要将Drawable转换成Bitmap

``` java
public static Bitmap drawableToBitmap(Drawable drawable) {
    Bitmap bitmap = Bitmap.createBitmap(
            drawable.getIntrinsicWidth(),
            drawable.getIntrinsicHeight(),
            drawable.getOpacity() != PixelFormat.OPAQUE ? Bitmap.Config.ARGB_8888
                    : Bitmap.Config.RGB_565);
    Canvas canvas = new Canvas(bitmap);
    //canvas.setBitmap(bitmap);
    drawable.setBounds(0, 0, drawable.getIntrinsicWidth(), drawable.getIntrinsicHeight());
    drawable.draw(canvas);
    return bitmap;
}
```

目前就了解到这么多，其他内容以后再更新吧。