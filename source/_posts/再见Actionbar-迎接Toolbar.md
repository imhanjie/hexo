---
title: '再见Actionbar,迎接Toolbar'
date: 2016-02-08 14:04:35
tags: [Android,Note]
---

Goodbye Actionbar , Welcome Toolbar

<!--more-->

> 本文参考自Android Developer原文 : [Adding the App Bar](http://developer.android.com/training/appbar/index.html)

#### **一、为什么要使用Toolbar而放弃使用Actionbar？**
从Android3.0开始(API level 11)，所有使用了默认主题的Activity都会将一个Actionbar作为app bar。许多app bar的功能都被添加到了Actionbar上，但是，随着Android版本的不断更新发布，造成了Actionbar在各种版本上的体验不一致，所以Google将这些Actionbar的功能全部放在support library内的Toolbar上，使用Toolbar代替Actionbar作为app bar。
值的注意的是，在使用Toolbar时，不推荐使用android.widget.Toolbar，推荐使用android.support.v7.widget.Toolbar，因为前者是在API Level 21被添加进来的，不能在低版本上使用(除非你的App的minsdk version为21)，而后者能向下最低兼容到Api Level 7。使用android.support.v7.widget.Toolbar另一个好处就是在Android 2.1 (API level 7) 上能体验到类似Material Design一样的体验。而如果Actionbar只能在运行在Android 5.0+才能体验到Material Design的效果。说了这么多，下面来介绍下Toolbar的基本使用(至于support库的下载以及配置这里就不多说了)

#### **二、向Activity内添加一个Toolbar作为App bar**

+ 1、让Activity继承自AppCompatActivity，因为AppCompatActivity对Toolbar有更好的支持，如果不想继承自AppCompatActivity而使用Toolbar，可能有点麻烦，可参考这篇文章 : [How to add Toolbar to an Activity which doesn’t extend AppCompatActivity](https://medium.com/google-developer-experts/how-to-add-toolbar-to-an-activity-which-doesn-t-extend-appcompatactivity-a07c026717b3#.hoxx62fs4)
+ 2、Toolbar和Actionbar不能同时存在，所以如果你想使用Toolbar来代替Actionbar，你需要让你的theme继承自Theme.AppCompat.Light.NoActionBar，或者直接在style中的app theme添加以下属性
``` java
<item name="windowNoTitle">true</item>
<item name="windowActionBar">false</item>
```
+ 3、添加Toolbar，将Toolbar当成一个普通的控件，例如Textview添加到layout xml布局文件中(最好放在布局文件中的最顶层)
``` java
<android.support.v7.widget.Toolbar
   android:id="@+id/my_toolbar"
   android:layout_width="match_parent"
   android:layout_height="?attr/actionBarSize"
   android:background="?attr/colorPrimary"
   android:elevation="4dp"
   app:theme="@style/ThemeOverlay.AppCompat.Dark.ActionBar"
   app:popupTheme="@style/ThemeOverlay.AppCompat.Light"/>
```
> 这里提下里面的两个属性：
> 1、android:elevation="4dp"：控制Toolbar的阴影(根据Material Design设计规范，Google推荐给Toolbar设置一个4dp的阴影效果就ok了)
> 2、app:popupTheme="@style/ThemeOverlay.AppCompat.Light"：指的是Toolbar右上角的弹出式菜单的theme

4、在Activity的onCreate()方法中
``` java
@Override
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.activity_my);
    Toolbar myToolbar = (Toolbar) findViewById(R.id.my_toolbar);
    setSupportActionBar(myToolbar);
    }
```
调用setSupportActionBar(myToolbar)直接将Toolbar作为activity的appbar，其实这里的setSupportActionBar是AppCompatActivity里面的方法，所以一开始我们最好的方式就是extends AppCompatActivity

下面看看运行的效果：
> ![1](https://raw.githubusercontent.com/bluehan/Images/master/ToolbarDemo/toolbar_1.png)


#### **三、给Toolbar添加菜单**

1、在res/menu目录下新建一个xml文件menu_main.xml，内容如下
``` java
<menu xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto">

    <item

        android:id="@+id/menu_call"
        android:icon="@android:drawable/ic_menu_call"
        android:orderInCategory="96"
        android:title="call"
        app:showAsAction="always" />

    <item
        android:id="@+id/menu_one"
        android:orderInCategory="96"
        android:title="menu one"
        app:showAsAction="never" />
    <item
        android:id="@+id/menu_two"
        android:orderInCategory="96"
        android:title="menu two"
        app:showAsAction="never" />
    <item
        android:id="@+id/menu_three"
        android:orderInCategory="96"
        android:title="menu three"
        app:showAsAction="never" />

</menu>
```

2、然后在onCreateOptionsMenu()中加载菜单：
``` java
@Override
    public boolean onCreateOptionsMenu(Menu menu) {
        MenuInflater menuInflater = getMenuInflater();
        menuInflater.inflate(R.menu.menu_main, menu);
        return super.onCreateOptionsMenu(menu);
    }
```
3、然后在onCreate()内设置处理菜单点击监听(这里只监听call button)
``` java
toolbar.setOnMenuItemClickListener(new Toolbar.OnMenuItemClickListener() {
            @Override
            public boolean onMenuItemClick(MenuItem item) {
                switch (item.getItemId()) {
                    case R.id.menu_call:{
                        Toast.makeText(MainActivity.this, "click call button", Toast.LENGTH_SHORT).show();
                        break;
                    }
                }
                return false;
            }
        });
```
下面看看运行效果：
> ![2](https://raw.githubusercontent.com/bluehan/Images/master/ToolbarDemo/toolbar_2.png) 


#### **四、给Toolbar左侧添加向上的按钮**

1、显示向上的按钮，在onCreate()中：
``` java
getSupportActionBar().setDisplayHomeAsUpEnabled(true);
```
看看运行图：
> ![3](https://raw.githubusercontent.com/bluehan/Images/master/ToolbarDemo/toolbar_3.png) 

2、处理向上的按钮的点击监听：

> 这里需要再添加一个SecondActivity作为MainActivity的父Activity，然后这只SecondActivity为启动APP时第一个显示的app，然后配置main activity的父activity为SecondActivity
``` java
<application
        android:allowBackup="true"
        android:icon="@mipmap/ic_launcher"
        android:label="@string/app_name"
        android:supportsRtl="true"
        android:theme="@style/AppTheme">
        <activity android:name=".SecondActivity">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />

                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>
        <activity
            android:name=".MainActivity"
            android:parentActivityName=".SecondActivity">

            <meta-data
                android:name="android.support.PARENT_ACTIVITY"
                android:value="hanjie.app.toolbardemo.SecondActivity" />
        </activity>
    </application>
```
**然后点击app启动SecondActivity，然后进入到MainActivity，点击左上角按钮，回到其父Acitvity SecondActivity。**


