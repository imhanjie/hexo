title: 理解Binder的工作机制
date: 2016-08-01 15:45:58
tags: [Android,Notes]
---

以上篇文章的客户端向服务端集合中添加元素并返回给客户端的例子，理解Binder的工作机制。

<!--more-->

---

[上篇文章](http://melodyxxx.com/2016/07/31/%E5%88%9D%E6%8E%A2AIDL/)讲了AIDL的基本使用方法，这篇文章主要讲AIDL的工作机制。

以上篇文章的客户端向服务端集合中添加元素并返回给客户端的例子，来看AIDL文件生成的对应的java文件：
``` java
/**
 * 接口文件，继承自IInterface
 */
public interface IMathAidlInterface extends android.os.IInterface {
    /**
     * 内部类Stub，继承自Binder，所以本质上是个Binder，运行在服务端
     */
    public static abstract class Stub extends android.os.Binder implements com.melodyxxx.aidlservice.IMathAidlInterface {
        private static final java.lang.String DESCRIPTOR = "com.melodyxxx.aidlservice.IMathAidlInterface";

        /**
         * Stub类构造器
         */
        public Stub() {
            this.attachInterface(this, DESCRIPTOR);
        }

        /**
         * 这个静态方法根据情况，如果客户端和服务端位于同一进程，则返回服务端Stub对象本身，如果客户端和服务端位于
         * 不同进程，则返回客户端Stub的代理Proxy
         */
        public static com.melodyxxx.aidlservice.IMathAidlInterface asInterface(android.os.IBinder obj) {
            if ((obj == null)) {
                return null;
            }
            android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
            if (((iin != null) && (iin instanceof com.melodyxxx.aidlservice.IMathAidlInterface))) {
                return ((com.melodyxxx.aidlservice.IMathAidlInterface) iin);
            }
            return new com.melodyxxx.aidlservice.IMathAidlInterface.Stub.Proxy(obj);
        }

        /**
         * 返回当前的Stub Binder对象
         */
        @Override
        public android.os.IBinder asBinder() {
            return this;
        }

        /**
         * 此方法运行在服务端，客户端代理Proxy调用mRemote.transact()时，系统会通过底层传输到服务端调用此方法，
         * 根据code来标识客户端调用的是哪个方法，reply用于服务端将数据回写传给客户端。
         */
        @Override
        public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags) throws android.os.RemoteException {
            switch (code) {
                case INTERFACE_TRANSACTION: {
                    reply.writeString(DESCRIPTOR);
                    return true;
                }
                case TRANSACTION_add: {
                    data.enforceInterface(DESCRIPTOR);
                    com.melodyxxx.aidlservice.Person _arg0;
                    if ((0 != data.readInt())) {
                        _arg0 = com.melodyxxx.aidlservice.Person.CREATOR.createFromParcel(data);
                    } else {
                        _arg0 = null;
                    }
                    java.util.List<com.melodyxxx.aidlservice.Person> _result = this.add(_arg0);
                    reply.writeNoException();
                    reply.writeTypedList(_result);
                    return true;
                }
            }
            return super.onTransact(code, data, reply, flags);
        }

        /**
         * Stub的内部代理类Proxy，若客户端和服务端不在同一进程，则客户端绑定服务端时，拿到了就是这个服务端的的代理Proxy对象
         */
        private static class Proxy implements com.melodyxxx.aidlservice.IMathAidlInterface {
            private android.os.IBinder mRemote;

            Proxy(android.os.IBinder remote) {
                mRemote = remote;
            }

            @Override
            public android.os.IBinder asBinder() {
                return mRemote;
            }

            public java.lang.String getInterfaceDescriptor() {
                return DESCRIPTOR;
            }

            /**
             * 代理类的add方法，给客户端调用，客户端调用后，将数据序列化到_data内，调用mRemote.transact，系统
             * 通过底层将数据传递到服务端，服务端的onTransact()方法会被调用，在onTransact()内将数据反序列化，
             * 在调用服务端的add方法，完成客户端远程调用服务端的方法的过程。reply则用于接收服务端返回的数据。
             */
            @Override
            public java.util.List<com.melodyxxx.aidlservice.Person> add(com.melodyxxx.aidlservice.Person aPerson) throws android.os.RemoteException {
                android.os.Parcel _data = android.os.Parcel.obtain();
                android.os.Parcel _reply = android.os.Parcel.obtain();
                java.util.List<com.melodyxxx.aidlservice.Person> _result;
                try {
                    _data.writeInterfaceToken(DESCRIPTOR);
                    if ((aPerson != null)) {
                        _data.writeInt(1);
                        aPerson.writeToParcel(_data, 0);
                    } else {
                        _data.writeInt(0);
                    }
                    mRemote.transact(Stub.TRANSACTION_add, _data, _reply, 0);
                    _reply.readException();
                    _result = _reply.createTypedArrayList(com.melodyxxx.aidlservice.Person.CREATOR);
                } finally {
                    _reply.recycle();
                    _data.recycle();
                }
                return _result;
            }
        }

        static final int TRANSACTION_add = (android.os.IBinder.FIRST_CALL_TRANSACTION + 0);
    }

    public java.util.List<com.melodyxxx.aidlservice.Person> add(com.melodyxxx.aidlservice.Person aPerson) throws android.os.RemoteException;
}
```

下面具体从客户端出发，具体分析下流程：

首先看客户端绑定服务端：
``` java
private ServiceConnection mConn = new ServiceConnection() {
    @Override
    public void onServiceConnected(ComponentName componentName, IBinder iBinder) {
        mIMathAidlInterface = IMathAidlInterface.Stub.asInterface(iBinder);
    }

    @Override
    public void onServiceDisconnected(ComponentName componentName) {

    }
};
```
onServiceConnected()内调用了服务端的Stub的asInterface方法，看asInterface方法：
``` java
public static com.melodyxxx.aidlservice.IMathAidlInterface asInterface(android.os.IBinder obj) {
    if ((obj == null)) {
        return null;
    }
    android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
    if (((iin != null) && (iin instanceof com.melodyxxx.aidlservice.IMathAidlInterface))) {
        return ((com.melodyxxx.aidlservice.IMathAidlInterface) iin);
    }
    return new com.melodyxxx.aidlservice.IMathAidlInterface.Stub.Proxy(obj);
}
```
如果客户端和服务端不在同一进程，则会return一个Stub的内部Proxy代理对象，代理对象Proxy也是实现了IMathAidlInterface接口的：
``` java
private static class Proxy implements com.melodyxxx.aidlservice.IMathAidlInterface {
	...
}
```

所以本例子中客户端绑定服务端时，拿到的并不是远程的Stub Binder对象，而是其代理类Proxy，然后客户端调用代理类对象mIMathAidlInterface的add方法向其添加元素，看代理类Proxy的内部add方法的实现：
``` java
@Override
public java.util.List<com.melodyxxx.aidlservice.Person> add(com.melodyxxx.aidlservice.Person aPerson) throws android.os.RemoteException {
    android.os.Parcel _data = android.os.Parcel.obtain();
    android.os.Parcel _reply = android.os.Parcel.obtain();
    java.util.List<com.melodyxxx.aidlservice.Person> _result;
    try {
        _data.writeInterfaceToken(DESCRIPTOR);
        if ((aPerson != null)) {
            _data.writeInt(1);
            aPerson.writeToParcel(_data, 0);
        } else {
            _data.writeInt(0);
        }
        mRemote.transact(Stub.TRANSACTION_add, _data, _reply, 0);
        _reply.readException();
        _result = _reply.createTypedArrayList(com.melodyxxx.aidlservice.Person.CREATOR);
    } finally {
        _reply.recycle();
        _data.recycle();
    }
    return _result;
}
```

首先从序列化池中中取出用于一个用于存放传递数据的Parcel对象_data和用于存放客户端返回数据的Parcel对象_reply，然后又创建了一个返回值类型List对象_result。然后可以看到将Person对象序列化到了_data中，然后调用了mRemote.transact()方法（mRemote对象就是远程服务内的Stub Binder对象），<font color='red'>**客户端挂起**</font>。transact()方法的第一参数是方法的标识，第二参数则是要传递的_data数据，第三个参数_reply用于接收服务端返回的数据。mRemote.transact()调用后，系统通过底层将刚才序列化的数据传递到了服务端，调用了**服务端的**onTransact()方法，看onTransact()内的实现：
``` java
@Override
public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags) throws android.os.RemoteException {
    switch (code) {
        case INTERFACE_TRANSACTION: {
            reply.writeString(DESCRIPTOR);
            return true;
        }
        case TRANSACTION_add: {
            data.enforceInterface(DESCRIPTOR);
            com.melodyxxx.aidlservice.Person _arg0;
            if ((0 != data.readInt())) {
                _arg0 = com.melodyxxx.aidlservice.Person.CREATOR.createFromParcel(data);
            } else {
                _arg0 = null;
            }
            java.util.List<com.melodyxxx.aidlservice.Person> _result = this.add(_arg0);
            reply.writeNoException();
            reply.writeTypedList(_result);
            return true;
        }
    }
    return super.onTransact(code, data, reply, flags);
}
```

可以看到根据方法标识code，调用对应的分支，这里是TRANSACTION_add即add方法，在第
行中，根据客户端传递过来的data数据反序列化除Person对象_arg0，然后在第十六行调用了服务端的add()方法，并将服务端add()方法返回的数据赋给了_result，接着在第18行将_result数据转化给了reply(这里的reply是客户端传递过来的用于接收服务端返回的数据)，然后return传递给了客户端。这时，<font color='red'>**客户端恢复继续执行**</font>，客户端根据_reply的数据反序列化出给_result，然后客户端返回_result给客户端的调用者。

至此，完成了客户端远程调用服务端的方法的过程。

---

**下面附上Binder的工作机制图：**

![](http://i.imgur.com/fAB1cPe.png)


---

**还需要注意：**

另外还需注意，客户端远程调用服务端的方法时，若服务端内发生异常，这里指的是除RemoteException以外的异常，不论发什么异常，**服务端是不会停止的**，但是部分异常能传递到客户端，例如NullPointerException空指针异常，有些异常不能传递到客户端，例如RuntimeException
。**若能传递到客户端，客户端就会崩溃停止运行，不能传递到客户端，客户端则无影响。**


