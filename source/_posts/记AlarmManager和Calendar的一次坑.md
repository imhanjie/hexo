---
title: 记AlarmManager和Calendar的一次坑
date: 2016-03-04 17:04:38
tags: [Android,Notes]
---

昨天在app内需要添加一个根据设置的早晚时间分别到点通知提示的一个功能，我的大致思路是开启一个Service，服务启动时获取到两个提示点的millis值，然后和当前的系统millis值比较，选取下一个最接近当前系统millis值的点来设置闹钟目标点。

<!--more-->

1、至于如何获取两个提示点的millis值，首先我这从配置文件获取的设置的时间格式为hour:minute(例如22:52)，然后提取出小时和分钟通过Calendar的set方法将小时和分钟设置进去，然后获取到该点的millis值来进行alarm的设置，获取millis值代码如下：
```java
Calendar calendar = Calendar.getInstance();
calendar.set(Calendar.HOUR_OF_DAY, Integer.parseInt(hour));
calendar.set(Calendar.MINUTE, Integer.parseInt(minute));
long targetMillis = calendar.getTimeInMillis();
```
2、然后根据targetMillis设置Alarm：
```java
Intent intent = new Intent(ACTION_REMIND_NOTIFICATION);
pendingIntent = PendingIntent.getBroadcast(RemindService.this, 0, intent, 0);
alarmManager.set(AlarmManager.RTC_WAKEUP, targetMillis, pendingIntent);
```
这里的pendingIntent是用来定时到点之后发送一个广播，收到广播后然后重新获取到两个提示点的millis值，然后和当前的系统millis值比较，选取下一个下一个最接近当前系统millis值的点，然后再重新设置Alarm，如此循环下去

然后就是测试.....

测试发现是能够到点提醒的，但是发现通知弹出后过了几秒又弹通知（通过通知的声音判断的），大概连续弹了好几次才消失，要是用户碰到这样，肯定骂人，疯狂弹通知。后来找原因，找了很久，才想到`Calendar.getInstance()`里面除了小时分钟，还有秒、毫秒值，所以当闹钟触发时，广播发送后，然后重复第1步，获取两个点的millis值，来和当前系统的millils比较获取下一个最近的millis值来设置闹钟，就是这里出了问题，当执行`Calendar.getInstance()`时，由于又重新到的是当前时间点的Calendar实例，里面的秒和毫秒字段是当前的值，所以不固定，而我要获取的两个点的millis值肯定是固定值（这里不考虑日期），不能每次获取时millis在变动，所以手动设置下`Calendar.SECOND`和`Calendar.MILLISECOND`的字段，代码调整如下：


```java
Calendar calendar = Calendar.getInstance();
calendar.set(Calendar.HOUR_OF_DAY, Integer.parseInt(hour));
calendar.set(Calendar.MINUTE, Integer.parseInt(minute));
calendar.set(Calendar.SECOND, 0); // 手动设置为0
calendar.set(Calendar.MILLISECOND, 0);  // 手动设置为0
long targetMillis = calendar.getTimeInMillis();
```

然后再测试.....

发现我的机器上在最近任务列表下不清除掉当前程序（Flyme5系统），一切正常，到点正确通知，且通知一次，然后到下一个点再提醒，功能完全正常。但是发现在最近任务列表下清除掉当前程序后，到点后发现pendingIntent没有发送出去，并且到点后居然重新调用了Service的onCreate()和onStartCommand()方法，后来在stackoverflow上找到了一个解决办法就是只要pendingIntent的requestCode不设置成0（原因暂不明确）,还要给intent加个flag,代码如下：
```java
intent.addFlags(Intent.FLAG_RECEIVER_FOREGROUND);
// 之前的
// pendingIntent = PendingIntent.getBroadcast(RemindService.this, 0, intent, 0);
// 修改后的
pendingIntent = PendingIntent.getBroadcast(RemindService.this, 1, intent, 0);
```

然后再测试，功能一切正常。

### 总结
* 如果要根据小时和分钟通过Calendar获取一个时间点固定的millis值（排除日期），不要忘记Calendar的秒和毫秒字段需要手动重置。
* 在使用AlarmManager的set方法时，为了保证pendingIntent能发送出去，pendingIntent的requestCode不要设置为0