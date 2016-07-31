title: "「Android-Art」理解Activity生命周期和启动模式以及IntentFilter匹配规则"
date: 2016-07-29 10:45:37
tags: [Android,Notes,Android开发艺术探索]
---

理解Activity生命周期和异常情况下的生命周期以及启动模式和IntentFilter匹配规则

<!--more-->

> 以下整理自「Android开发艺术探索」第一章

---

## 一、Activity的生命周期全面分析
<div align = center>
![](http://i.imgur.com/ivyk9lo.png)

上图来自Androi官方文档
</div>

### 1、典型情况下的生命周期分析
- **onCreate:** 表示Activity正在被创建，这是生命周期的第一个方法。在这个方法中，我们可以做一些初始化工作，比如调用setContentView去加载界面布局资源、初始化Activity所需数据等。
- **onRestart:** 表示Activity正在重新启动。一般情况下，当当前Activity从不可见重新变为可见状态时，onRestart就会被调用。
- **onStart:** 表示Activity正在被启动，即将开始，这是Activity已经可见了，但是还没有出现在前台，还无法和用户交互。这时候其实可以理解为Activity已经显示出来了，但是我们还看不到
- **onResume:** 表示Activity已经可见了，并且出现在前台并开始活动。需要注意和onStart的对比，onStart和onResume都表示Activity已经可见了，但是onStart的时候Activity还在后台，onResume的时候Activity才显示到前台
- **onPause:** 表示Activity正在停止，正常情况下，紧接onStop就会被调用。在特殊情况下，如果这个时候快速地再回到当前Activity，那么onRrsume会被调用(属于极端情况，用户操作很难重现这一场景)。此时可以做一些存储数据、停止动画等工作，但是注意**不能太耗时，因为这会影响到新Activity的显示，onPause必须先执行完，新Activity的onResume才会执行**
- **onStop:** 表示Activity即将停止，可以做一些稍微重量级的回收工作，同样不能太耗时
- **onDestroy:** 表示Activity即将被销毁，这是Activity生命周期中的最后一个回调，在这里，我们可以做一些回收工作和最终资源的释放。


**或许还要知道**：
- 如果新Activity采用了透明主题，那么当前Activity不会回调onStop
- onStart和onStop是从Activity是否可见这个角度来回调的，而onResume和onPause是从Activity是否位于前台这个角度来回调的，除了这种区别，在实际使用过程中没有其他明显的区别
- AMS内部维护着一个ActivityStack并负责栈内的Activity状态的同步
- 旧Activity先onPause，然后新的Activity再启动

### 2、异常情况下的生命周期分析

** 情况1：资源相关的系统配置发生改变导致Activity被杀死并重新创建 **

当系统配置发生改变后，Activity会被销毁，其onPause、onStop、onDestroy均会被调用，同时由于Activity是在异常情况下终止的，系统会调用onSaveInstanceState来保存当前Activity的状态。这个方法的调用时机是在onStop之前，他和onPause没有既定的时序关系，它既可能在onPause之前调用，也可能在onPause之后调用。当Activity被重新创建后，系统会调用onRestoreInstanceState，并且把Activity销毁时onSaveInstanceState方法保存的Bundle对象作为参数同时传递给onRestoreInstanceState和onCreate方法。因此我们可以通过onCreate和onRestoreInstanceState方法判断Activity是否被重建了(看bundle是否为null)，从时序上来说，onRestoreInstanceState的调用时机在onStart之后。

保存数据的时候，系统会默认为我们保存当前Activity的视图结构，并且在Activity重启后为我们恢复这些数据，比如文本框中用户输入的数据等等，查看源码可以发现，View内也有onSaveInstanceState方法。系统采用的是委托的思想，Activity调用onSaveInstanceState去保存数据，然后Activity会委托Window去保存数据，接着Window会委托它上面的顶层容器去保存数据。顶层容器是一个ViewGroup，一般来说它很可能是DecorView。最后顶层容器再一一通知它的子元素来保存数据。View的绘制、事件分发等过程中都采用类似的思想，具体的View的onSaveInstanceState实现都是不一样的。

> **这里要注意下，书中说的onSaveInstanceState调用时机有点问题，它不仅在Activity异常终止情况下会调用，而且在系统认为此Activity可能会被系统回收时也会调用该方法，例如锁屏，切回后台，都会在onStop之前调用onSaveInstanceState方法，这点需要注意。**

** 情况2：资源内存不足导致低优先级的Activity被杀死 **

这种情况不好模拟，但是其数据存储和恢复过程和情况1完全一致。

如果当某项内容发生改变后，我们不想系统重新创建Activity，可以给Activity指定configChanges属性。比如不想让Activity在屏幕旋转的时候重新创建就可以给configChanges属性添加orientation这个值
``` xml
android:configChanges="orientation"
android:configChanges="orientation|screenSize"  //add "|screenSize" to configChanges when targeting API level 13+
```
此时当Activity在旋转屏幕时就不会重新创建Activity了，会转而调用Activity的
``` java
public void onConfigurationChanged(Configuration newConfig);
```


## 二、Activity的启动模式

- **standard:** 标准模式，也是默认的启动模式。每次启动一个Activity都会重新创建一个新的实例。谁启动了这个Activity，那么这个Activity就运行在启动它的那个Activity所在的栈中，所有当你用ApplicationContext去启动standard模式的Activity的时候就会报错，因为非Activity类型的Context(如ApplicationContext)并没有所谓的任务栈，所以就会有问题，只要为待启动Activity指定FLAG_ACTIVITY_NEW_TASK标记位，这样在启动的时候回创建一个新的任务栈，这个时候启动的Activity实际上是以singleTask模式启动的。
- **singleTop:** 栈顶复用模式。在这种模式下，如果新Activity已经位于任务栈的栈顶，那么此Activity不会被重新创建，同时它的onNewIntent方法会被调用(完整过程的是: onPause()->onNewIntent()->onResume())，通过此方法的参数我们可以取出当前请求的信息。如果新Activity的实例已经存在但不是位于栈顶，那么新Activity仍然会重新创建。
- **singleTask:** 栈内复用模式。当一个具有singleTask模式的Activity请求启动后，比如Activity A，系统会首先寻找是否存在A想要的任务栈，如果不存在，就重新创建一个任务栈，然后创建A的实例后把A放入栈中。如果存在A所需的任务栈，这时要看A是否在栈中有实例存在，如果有实例存在，那么系统就会把A调到栈顶(并清除它以上的所有Activity)并调用它的onNewIntent方法完整过程的是: onPause()->onNewIntent()->onResume())，如果实例不存在，就把创建A的实例并把A压入栈中。
- **singleInstance:** 单实例模式。加强的singleTask模式，它除了具有singleTask模式的所有特性wait，还加强了一点，那就是具有此模式的Activity只能单独地位于一个任务栈中。

上面的singleTask模式中提到"Activity所需的任务栈"，这里涉及到Activity的TaskAffinity属性，这个属性的默认值为应用的包名，可以为TaskAffinity属性指定一个值(字符串)，这个值即为此Activity所需任务栈的名字。TaskAffinity属性主要和singleTask启动模式或者allowTaskReparent属性配对使用，在其他情况下没有意义。(可以对照书上P19-P20多理解理解，容易忘)

## 三、IntentFilter匹配规则

> 隐式调用需要Intent能够匹配目标组件的IntentFilter中所设置的过滤信息，如果不匹配将无法启动目标Activity

** 1、先了解一下匹配的概念 **

IntentFilter中的过滤信息有action、category、data。为了匹配过滤列表(xml中的过滤规则)，Intent需要同时匹配过滤列表中的action、category、data信息，否则匹配失败，一个过滤列表中的action，category和data可以有多个，同一类别的信息共同约束当前类别的匹配过程。只有一个Intent同时匹配action类别、category类别、data类别才能算完全匹配，只有完全匹配才能成功启动目标Activity。另外一点，一个Activity中可以有多个intent-filter，一个Intent只要能够匹配任何一组intent-filter即可成功启动对应的Activity。

** 2、再来了解匹配的规则 **

**action匹配规则：** 如果规则中有action，Intent必要有一个或多个与其匹配
**category匹配规则：** Intent可以没有category，但是如果有，必须是在规则中定义过了的，另外系统默认会在Intent内加上`android.intent.category.DEFAULT`，所以根据匹配规则，xml中的规则也要加上`android.intent.category.DEFAULT`，否则无法匹配成功。
**data匹配规则：**见下面

** 2、单独说说data的匹配规则 **
了解data匹配规则之前，先看看data的结构，因为data稍微有点复杂
``` xml
<data
	android:scheme="string"
	android:host="string"
	android:port="string"
	android:path="string"
	android:pathPattern="string"
	android:pathPrefix="string"
	android:mimeType="string" />
```

data由两部分组成，mimeType和URI，mimeType对应android:mimeType="string"部分，剩余部分对应URI部分。mimeType指媒体类型，比如image/jpeg、audio/mpeg4-generic和video/*等，可以表示图片、文本、视频等不同的媒体格式，而URI中包含的数据就比较多了，下面是URI的结构：
```
<scheme>://<host>:<port>/[<path>|<pathPrefix>|<pathPattern>]
```
例如：
```
content://com.example.project:200/folder/subfolder/etc
http://www.melodyxxx.com:80/search/info
```

- **scheme:** URI的模式，比如http、file、content等，如果URI中没有指定scheme，那么整个URI的其他参数无效，这也意味着URI是无效的
- **host:** URI的主机名，比如www.melodyxxx.com，如果host未指定，那么整个URI中的其他参数无效，这也意味着URI是无效的
- **port:** URI中的端口号，比如80，仅当URI中指定了scheme和host参数的时候port参数才是有意义的
- **path、pathPattern和pathPrefix:** 这三个参数表述路劲信息，其中path表示完整的路径信息；pathPattern也表示完整的路径信息，但是它里面可以包含通配符"\*","\*"表示0个或多个任意符号，需要注意的是，由于正则表达式的规范，如果想表示真实的字符串，那么"\*"要写成"\\\\*"，"\"要写成"\\\\\\\\";pathPrefix表示路径的前缀信息。

**data的匹配规则:** 和action类似，他也要求Intent中必须含有data数据，并且data数据能够完全匹配过滤规则中的某一个data

**IntentFilter的匹配规则对于BroadcastReceiver和Service也是同样的道理。**