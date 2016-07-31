title: 「Android-Art」理解PendingIntent
date: 2016-07-25 16:42:44
tags: [Android,Notes,Android开发艺术探索]
---

理解PendingIntent，理解PendingIntent的flag参数

<!--more-->

### 以下整理自「Android开发艺术探索」第五章 ###

---



### PendingIntent和Intent简单比较 ###
- PendingIntent表示一种处于待触发、待定、即将发生的Intent
- 而Intent是立刻发生的


### PendingIntent的典型使用场景 ###
- 给RemoteViews添加点击事件

### PendingIntent支持三种待定意图 ###
- 启动Activity
- 启动Service
- 发送广播


|返回值|方法及其作用|
|-|-|
|static PendingIntent|getActivity(Context context,int requestCode,Intent intent,int flags)获得一个PendingIntent，该待定意图发生时，效果相当于Context.startActivity(Intent)|
|static PendingIntent|getService(Context context,int requestCode,Intent intent,int flags)获得一个PendingIntent，该待定意图发生时，效果相当于Context.startService(Intent)|
|static PendingIntent|getBroadcast(Context context,int requestCode,Intent intent,int flags)获得一个PendingIntent，该待定意图发生时，效果相当于Context.sendBroadcast(Intent)|

参数列表：
- context: 上下文对象
- requestCode: 请求码，会影响到flags的效果
- intent: intent意图
- flags: 常见类型有
	- FLAG_ONE_SHOT
	- FLAG_CANCEL_CURRENT
	- FLAG_UPDATE_CURRENT

在解释这3个标记位之前，首先要明白PendingIntent的匹配规则，即在什么情况下PendingIntent是相同的。

**PendingIntent的匹配规则为：**
- 如果两个PendingIntent他们内部的Intent相同并且requestCode也相同，那么这两个PendingIntent就是相同的，requestCode好理解，那什么情况下Intent相同呢？

**Intent的匹配规则为：**
- 如果两个Intent的ComponentName(就是Intent常用的2个参数的构造器的第二个参数xxx.class构造来的)和intent-filter都相同，那么这两个Intent就是相同的。需要注意的是Extras不参与Intent的匹配过程，只要Intent之间的ComponentName和intent-filter相同即可。


### 3个标记位的区别？###

举个例子说明，当我们在发送多个通知Notification的时候，调用NotificationManager的notify(id,notificaiton);
- 此时如果**每个通知的id都一样**，那么**不管PendingIntent是否匹配，后面的通知都会直接替换前面的通知**
- 此时如果**每个通知的id都不同**，这里再分两种情况
	- **PendingIntent不匹配时**，这种情况下无论使用哪种标记位，这些通知都不会相互干扰的。
	- **PendingIntent匹配时**，这里标记位起作用了：
		- **FLAG_ONE_SHOT**: 后续的通知中的PendingIntent会和第一条通知保持完全一致，包括其中的Extras，**单击任何一条通知后，剩下的通知均无法打开**，当所有的通知都被清除后，会再次重复这个过程
		- **FLAG_CANCEL_CURRENT**: **只有最新的通知可以打开**，之前弹出的所有通知均无法打开
		- **FLAG_UPDATE_CURRENT**: **之前弹出的通知中的PendingIntent会被更新，最终他们和最新的一条通知保持完全一致**，包括启动的Extras，并且这些通知都是可以打开的

---