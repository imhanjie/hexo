title: 「Android-Art」理解RemoteViews
date: 2016-07-24 22:42:44
tags: [Android,Notes,Android开发艺术探索]
---

RemoteVies在自定义通知栏布局和桌面Widget的开发中扮演着重要的角色

<!--more-->

### 以下整理自「Android开发艺术探索」第五章 ###

---

RemoteViews顾名思义，远程View，RemoteViews表示的是一个View结构，他可以在其他进程中显示，由于他在其他进程中显示，为了能够更新它的界面。RemoteViews提供了一组基础的操作用于跨进程更新它的界面。自定义通知栏布局和桌面Widget更新界面时无法像Activity里面那样去直接更新View，这是因为他们的界面运行在其他进程中，确切来说是系统的SystemServer进程。为了提供更新界面，RemoteViews提供了一系列的set方法。


最常用的构造方法：
``` java
public RemoteViews(String packageName , int layoutId)
```
第一个参数是包名，第二个参数是布局文件的资源id;

**注意：RemoteView并不支持所有的View，只支持部分View(因为更新UI是通过多进程的，所以出于性能考虑)**

* 下面列举了一些常用的set方法：

|方法名|作用|
|---|---|
|setTextViewText(int viewId, CharSequence text)|设置TextView的文本|
|setTextViewTextSize(int viewId, int units, float size)|设置TextView的字体大小|
|setTextColor(int viewId, int color)|设置TextView的字体颜色|
|setImageViewResource(int viewId, int srcId)|设置ImageView的图片资源|
|setInt(int viewId, String methodName, int value)|反射调用View对象的参数类型为int的方法|
|setLong(int viewId, String methodName, long value)|反射调用View对象的参数类型为long的方法|
|setBoolean(int viewId, String methodName, boolean value)|反射调用View对象的参数类型为boolean的方法|
|setOnClickPendingIntent(int viewId, PendingIntent pendingIntent)|为View添加单击事件，事件类型只能为PendingIntent|

### RemoteView的内部机制 ###

- 首先RemoteViews会通过**Binder**传递到SystemSever进程，这是因为RemoteViews实现了Parcelable接口，因此可以跨进程传输
- 系统会根据RemoteViews中的包名等信息去得到该应用的资源。
- 通过LayoutInflater去加载RemoteViews中的布局文件
- 接着系统会对View执行一系列界面更新任务，这些任务就是之前我们通过set方法来**提交**的。set方法对View的操作不是立刻执行的，在RemoteViews内部会记录所有的更新操作，等到RemoteViews被加载以后才能执行


![](http://i.imgur.com/YNbpXgK.png)


理论上，系统完全可以通过Binder去支持所有的View和View操作，但这样做代价太大，因为View的方法太多了，另外就是大量的IPC操作会影响效率。为了解决这个问题，**系统并没有通过Binder去直接支持View的跨进程访问，而是提供了一个Action的概念，Action代表一个View的操作，Action同样实现了Parcelable接口。系统首先将View操作封装到Action对象并将这些对象跨进程传输到远程进程，接着远程进程中执行Action对象中的具体操作。**


例如看setTextViewText方法的实现：
``` java
public void setTextViewText(int viewId, CharSequence text) {
    setCharSequence(viewId, "setText", text);
}
```

``` java
public void setCharSequence(int viewId, String methodName, CharSequence value) {
    addAction(new ReflectionAction(viewId, methodName, ReflectionAction.CHAR_SEQUENCE, value));
}
```

``` java
private void addAction(Action a) {

	...   

    if (mActions == null) {
        mActions = new ArrayList<Action>();
    }
    mActions.add(a);

    ...
}
```

可以看到RemoteViews内部有个mActions成员，用于存储每次set的Action的操作。

``` java
public View apply(Context context, ViewGroup parent, OnClickHandler handler) {
    RemoteViews rvToApply = getRemoteViewsToApply(context);

    View result;
    
	...

    LayoutInflater inflater = (LayoutInflater)
            context.getSystemService(Context.LAYOUT_INFLATER_SERVICE);

    // Clone inflater so we load resources from correct context and
    // we don't add a filter to the static version returned by getSystemService.
    inflater = inflater.cloneInContext(inflationContext);
    inflater.setFilter(this);
    result = inflater.inflate(rvToApply.getLayoutId(), parent, false);

    rvToApply.performApply(result, parent, handler);

    return result;
}
```

``` java
private void performApply(View v, ViewGroup parent, OnClickHandler handler) {
    if (mActions != null) {
        handler = handler == null ? DEFAULT_ON_CLICK_HANDLER : handler;
        final int count = mActions.size();
        for (int i = 0; i < count; i++) {
            Action a = mActions.get(i);
            a.apply(v, parent, handler);
        }
    }
}
```

RemoteViews的apply方法是来进行View的更新操作的。可以看到，RemoteViews的apply方法内部会遍历调用所有Action对象的apply方法，具体的View更新操作是由Action对象的apply方法来完成的。
至于apply是怎么被调用的，例如当我们在小部件开发中调用AppWidgetManager或者通知栏的NotificationManager的notify方法，他们的确是通过RemoteViews的apply以及reapply方法来加载或者更新界面的。

apply和reapply的区别在于：
- apply会加载布局并更新界面
- reApply则只会更新界面

通知栏和桌面小插件在初始化界面时会调用apply方法，而在后续的更新界面时则会调用reapply方法。

下面看一些Action的子类的具体实现：

- ReflectionAction

``` java
private final class ReflectionAction extends Action {

	...

    ReflectionAction(int viewId, String methodName, int type, Object value) {
        this.viewId = viewId;
        this.methodName = methodName;
        this.type = type;
        this.value = value;
    }

	...

    @Override
    public void apply(View root, ViewGroup rootParent, OnClickHandler handler) {
        final View view = root.findViewById(viewId);
        if (view == null) return;

        Class<?> param = getParameterType();
        if (param == null) {
            throw new ActionException("bad type: " + this.type);
        }

        try {
            getMethod(view, this.methodName, param).invoke(view, wrapArg(this.value));
        } catch (ActionException e) {
            throw e;
        } catch (Exception ex) {
            throw new ActionException(ex);
        }
    }
	
	...

}
```

ReflectionAction表示的是一个反射动作。使用ReflectionAction的set方法有：setTextViewText、setBoolean、setLong、setDouble等等。

除了ReflectionAction还有其他的Action，下面来看看TextViewSizeAction：

- TextViewSizeAction

``` java
private class TextViewSizeAction extends Action {
    public TextViewSizeAction(int viewId, int units, float size) {
        this.viewId = viewId;
        this.units = units;
        this.size = size;
    }

	...

    @Override
    public void apply(View root, ViewGroup rootParent, OnClickHandler handler) {
        final TextView target = (TextView) root.findViewById(viewId);
        if (target == null) return;
        target.setTextSize(units, size);
    }

    public String getActionName() {
        return "TextViewSizeAction";
    }

    int units;
    float size;

    public final static int TAG = 13;
}
```

TextViewSizeAction的实现比较简单，他之所以不用反射来实现，是因为setTextSize这个方法有2个参数，因此无法复用ReflectionAction，因为ReflectionAction反射调用只有一个参数。



### 以上整理自「Android开发艺术探索」第五章 ###