title: 使用Notification的BigTextStyle时需要注意的地方
date: 2016-03-16 23:12:59
tags: [Android,Notes]
---

Android中使用Notification时，如果遇到contentText文本过长时，一行无法显示全的话，可以给notification设置一个style--BigTextStyle，让其contentText部分自动拓展多行显示，但是在使用的时候需要注意一些东西

<!--more-->

先贴出Notification中使用BigTextStyle方法

``` java
        NotificationCompat.Builder builder = new NotificationCompat.Builder(this);
        builder.setSmallIcon(R.drawable.ic_weather_tip)
                .setTicker(tickerText)
                .setContentTitle(titleText)
                .setContentText(contentText)
                .setStyle(new NotificationCompat.BigTextStyle()
                        .bigText(yourLongText))
                .setSound(alarmSound)
                .setAutoCancel(true)
                .setContentIntent(pendingIntent);
        NotificationManager notificationManager = (NotificationManager) getSystemService(Context.NOTIFICATION_SERVICE);
        notificationManager.notify(id, builder.build());
```
**关于contentTitle**

* 如果builder中设置`setContentTitle()`并且在BigTextStyle中也设置了`setBigContentTitle()`，那么后者将会代替前者

**关于contentText**

> 在builder中设置是用setContentText()设置的
> 在BigTextStyle中设置是用bigText()设置的


有以下2种情况bigTextStyle.bigText()会失效：

* 如果builder中设置的内容不长，一行能显示的下时，BigTextStyle中设置了也没用
* 如果builder中设置的内容很长，但BigTextStyle中设置的不长(即一行能显示的下)，那么后者仍失效，会照常显示builder中设置的内容(即后面显示不下带省略号)


