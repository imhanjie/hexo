title: Messenger的使用方法以及源码分析
date: 2016-08-02 11:18:31
tags: [Android,Notes]
---

Messenger是一种轻量级的IPC解决方案，可用于进程间的简单的数据传递。

<!--more-->

Messenger的主要作用是**跨进程传递消息**，但要直接跨进程调用服务端的方法，那么用Messenger就无法办到。因为Messenger会在单一线程中创建包含所有请求的队列，所以不必对服务进行线程安全设计。

本文主要了解：
- **Messenger的基本使用方法**
- **Messenger传递自定义类型数据时需要注意的地方**
- **Messenger的实现原理(基于AIDL)**
- **总结**

---

### 一、Messenger的基本使用方法

使用Messenger也是很简单的：

首先编写服务端：

``` java
public class MessengerService extends Service {

    public static final int MSG_CLIENT_TO_SERVICE = 0;

    // Hanler用于接收客户端发来的消息
    class MessengerHandler extends Handler {
        @Override
        public void handleMessage(Message msg) {
            switch (msg.what) {
                case MSG_CLIENT_TO_SERVICE:
                    Bundle data = msg.getData();
                    String info = data.getString("info");
                    Log.i("bingo", "Message from client: " + info);
                    break;
                default:
                    super.handleMessage(msg);
            }
        }
    }

    // 提供给客户端用来让客户端发消息用的
    private Messenger mMessenger = new Messenger(new MessengerHandler());

    @Override
    public IBinder onBind(Intent intent) {
        return mMessenger.getBinder();
    }

}
```

服务端MessengerService内定义了一个MessengerHandler用来处理客户端发来的消息，这里当收到客户端的消息时打印客户端中所携带的数据。。然后创建一个Messenger对象用来提供给客户端，使用MessengerHandler作为构造参数，然后当客户端绑定服务端的时候，返回Messenger的IBinder接口给客户端，客户端拿到此IBinder的时候，就可以用这个接口再构造出一个Messenger，然后用此Messenger来给服务端发送消息。

然后看客户端：

``` java
public class MainActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        Intent bindIntent = new Intent(this, MessengerService.class);
        bindService(bindIntent, mConn, Context.BIND_AUTO_CREATE);
    }

    private ServiceConnection mConn = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName componentName, IBinder iBinder) {
            Messenger service = new Messenger(iBinder);
            Message msg = Message.obtain();
            msg.what = MessengerService.MSG_CLIENT_TO_SERVICE;
            Bundle data = new Bundle();
            data.putString("info", "I am Client!");
            msg.setData(data);
            try {
                service.send(msg);
            } catch (RemoteException e) {
                e.printStackTrace();
            }
        }

        @Override
        public void onServiceDisconnected(ComponentName componentName) {

        }
    };

}
```

也比较简单，Activity创建时绑定服务，在onServiceConnected内使用IBinder参数创建一个Messenger，利用这个Messenger即可向服务端发送数据。这里发送一个String对象过去，调用messenger的send方法将Message发送出去。

然后注册一下Service：
``` xml
<service
    android:name=".MessengerService"
    android:process=":remote"/>
```

然后运行程序，查看打印的log：
```
08-02 23:24:22.746 25156-25156/com.melodyxxx.messengertest:remote I/bingo: Message from client: I am Client!
```

服务端成功接收到客户端发送的消息，并打印了log。

上面演示的只是单向的客户端向服务端发送数据，那么当服务端接收到消息时怎么向客户端回复消息呢？也很简单，在客户端也定义一个Handler来处理服务端发送的消息，然后像上面服务端一样，使用此Handler创建一个Messenger对象，作为message的replyTo传递给服务端，然后服务端收到消息后，顺便取出此Messenger即可使用其向客户端发送数据：

客户端：

``` java
public class MainActivity extends AppCompatActivity {

    public static final int MSG_SERVICE_TO_CLIENT = 1;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        Intent bindIntent = new Intent(this, MessengerService.class);
        bindService(bindIntent, mConn, Context.BIND_AUTO_CREATE);
    }

    class MessengerHandler extends Handler {
        @Override
        public void handleMessage(Message msg) {
            switch (msg.what) {
                case MSG_SERVICE_TO_CLIENT:
                    Bundle data = msg.getData();
                    String reply = data.getString("reply");
                    Log.i("bingo", "Message from service: " + reply);
                    break;
            }
            super.handleMessage(msg);
        }
    }

    private Messenger mMessenger = new Messenger(new MessengerHandler());

    private ServiceConnection mConn = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName componentName, IBinder iBinder) {
            Messenger service = new Messenger(iBinder);
            Message msg = Message.obtain();
            msg.what = MessengerService.MSG_CLIENT_TO_SERVICE;
            Bundle data = new Bundle();
            data.putString("info", "I am Client!");
            msg.setData(data);
            msg.replyTo = mMessenger;
            try {
                service.send(msg);
            } catch (RemoteException e) {
                e.printStackTrace();
            }
        }

        @Override
        public void onServiceDisconnected(ComponentName componentName) {

        }
    };

}
```

服务端：
``` java
public class MessengerService extends Service {

    public static final int MSG_CLIENT_TO_SERVICE = 0;

    // Hanler用于接收客户端发来的消息
    class MessengerHandler extends Handler {
        @Override
        public void handleMessage(Message msg) {
            switch (msg.what) {
                case MSG_CLIENT_TO_SERVICE:
                    Bundle data = msg.getData();
                    String info = data.getString("info");
                    Log.i("bingo", "Message from client: " + info);
                    // 向客户端回复消息
                    Messenger client = msg.replyTo;
                    Message replyMessage = Message.obtain();
                    replyMessage.what = MainActivity.MSG_SERVICE_TO_CLIENT;
                    Bundle replyData = new Bundle();
                    replyData.putString("reply", "Hello , I am Service!");
                    replyMessage.setData(replyData);
                    try {
                        client.send(replyMessage);
                    } catch (RemoteException e) {
                        e.printStackTrace();
                    }
                    break;
                default:
                    super.handleMessage(msg);
            }
        }
    }

    // 提供给客户端用来让客户端发消息用的
    private Messenger mMessenger = new Messenger(new MessengerHandler());

    @Override
    public IBinder onBind(Intent intent) {
        return mMessenger.getBinder();
    }

}
```

然后运行，查看log：
```
08-02 23:42:57.021 8219-8219/com.melodyxxx.messengertest:remote I/bingo: Message from client: I am Client!
08-02 23:42:57.041 8190-8190/com.melodyxxx.messengertest I/bingo: Message from service: Hello , I am Service!
```

---

### 二、Messenger传递自定义类型数据时需要注意的地方

Messenger传递我们自定义的类型时，**需要将对象序列化**，然后使用Bundle的putParcelable()方法放入Bundle中，然后需要特别注意，**在服务端接收的时候，需要调用Bundle的setClassLoader()方法指定类加载器，否则在Bundle的getParcelable()取出对象的时候，会提示类找不到，这点尤其需要注意。**

还有也可以使用Message的obj参数传递对象，但是只能传递系统提供的实现了Parcelable接口的对象，不能传递我们自定义的类型的对象。

---

### 三、Messenger的实现原理(基于AIDL)

Messenger有两个构造方法,首先看第一个：

``` java
public Messenger(Handler target) {
    mTarget = target.getIMessenger();
}
```

这个构造方法需要一个Handler，就是上面的例子在服务端中使用到利用Handler构造一个Messenger对象。先看mTarget的类型：
``` java
private final IMessenger mTarget;
```
是一个IMessenger接口，接着看hanlder的getIMessemger实现：

``` java
final IMessenger getIMessenger() {
    synchronized (mQueue) {
        if (mMessenger != null) {
            return mMessenger;
        }
        mMessenger = new MessengerImpl();
        return mMessenger;
    }
}
```

这里创建了一个MessengerImpl对象，MessengerImpl是继承自IMessenger.Stub，明显是AIDL的实现方法。MessengerImpl也是Handler的内部类：
``` java
private final class MessengerImpl extends IMessenger.Stub {
    public void send(Message msg) {
        msg.sendingUid = Binder.getCallingUid();
        Handler.this.sendMessage(msg);
    }
}
```

所以可以知道了，上面根据Handler创建出来的Messenger，Messenger的内部的mTarget为
MessengerImpl，也就是一个IMessenger.Stub，然后客户端在绑定服务端时调用了Messenger的getBinder()方法：

``` java
@Override
public IBinder onBind(Intent intent) {
    return mMessenger.getBinder();
}
```

``` java
public IBinder getBinder() {
    return mTarget.asBinder();
}
```
可以getBinder()内调用了mTarget即IMessenger.Stub的asBinder()方法，之前的文章也介绍了AIDL的使用和Binder的工作机制，可以知道asBinder返回的就是IMessenger.Stub自己，因为IMessenger.Stub也是继承自Binder的。IMessenger.Stub运行在服务端内的，所以由此分析可以客户端绑定服务端时将IMessenger.Stub即IBinder接口返回给客户端了。

然后接下来看客户端的分析：

根据上面的Messenger使用方法，在onServiceConnected()内：
``` java
        @Override
        public void onServiceConnected(ComponentName componentName, IBinder iBinder) {
            Messenger service = new Messenger(iBinder);

            ...

            try {
                service.send(msg);
            } catch (RemoteException e) {
                e.printStackTrace();
            }
        }
```
根据服务端传过来的IBinder接口(IMessenger.Stub)构造出了一个在客户端使用的Messenger，看这个Messenger的构造方法：
``` java
public Messenger(IBinder target) {
    mTarget = IMessenger.Stub.asInterface(target);
}
```
分析到这里，已经完全可以可知道Messenger就是内部封装了AIDL的实现，在这个构造方法内的调用了IMessenger.Stub.asInterface(target)，在上一篇Binder的工作机制中，可以知道这个方法是分情况的，当客户端和服务端运行在同一进程中时，返回的就是Stub本身，若客户端和服务端不在同一进程中时，返回的是Stub的一个内部类代理Proxy。这里客户端和服务端不在同一进程中时，返回的是Stub的一个内部类代理Proxy给了mTarget，然后调用这个Messenger发送消息，看Messenger的send()实现：
``` java
public void send(Message message) throws RemoteException {
    mTarget.send(message);
}
```
明显调用mTarget的send，而mTarget是一个远程代理，所以Binder的工作机制可以知道，通过底层然后调用服务端间接调用了服务端的Handler发送消息，然后服务端的Handler接收消息并处理消息。

---

### 总结：
- Messenger是一种轻量级的IPC机制
- Messenger的主要作用是**跨进程传递消息**
- Messenger内部是通过AIDL实现的
- Messenger在传递自定义类型时，取出时记得给Bundle设置正确的ClassLoader
- Messenger是执行进程间通信 (IPC) 的最简单方法，因为 Messenger 会在单一线程中创建包含所有请求的队列，这样您就不必对服务进行线程安全设计


> 参考资料： 「Android开发艺术探索」 & Android官方文档

