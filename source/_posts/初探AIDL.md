title: 初探AIDL
date: 2016-07-31 13:33:31
tags: [Android,Notes]
---

了解AIDL的使用方法

<!--more-->

---

概要：
1. 介绍一个AIDL的Demo
2. 传递自定义类型

---

Demo介绍：
包含服务端和客户端，客户端通过远程绑定服务去调用服务端的方法，服务端再把数据返回给客户端，IDE使用的是Android Studio。

## 一、Demo

### 服务端

##### 1、创建一个名为AIDLService的project作为服务端
##### 2、创建aidl文件，定义接口
![](http://i.imgur.com/NtPb62S.png)

![](http://i.imgur.com/2R5Bdyn.png)

![](http://i.imgur.com/VLqJUmV.png)

创建一个名为**IMathAidlInterface**的aidl文件，在Android Studio中直接创建aidl文件会自动帮我们创建AIDL文件夹以及package。

Android Studio自动为**IMathAidlInterface**文件填充的模板如下:

![](http://i.imgur.com/mm6zwux.png)

其中那个接口内的自动出现的方法是展示可以使用的基本类型，包括所有的基本数据类型（short除外）、String、CharSequence、List、Map、Parcelable

这里可以删去，定义我们自己计算接口，这里定义一个加法接口。

![](http://i.imgur.com/xDTvKai.png)

这里注意，接口名IMathAidlInterface要和aidl文件名要保持一致。

至此AIDL文件创建完毕，这里介绍一下AIDL文件的作用：
- AIDL全称Android Interface Definition Language (AIDL)，Android接口定义语言。利用这个文件就可以生成一个对应的Java文件。在Eclipse中，编写完AIDL文件，类似R文件，会自动在gen目录下生成一个AIDL对应的Java文件，但在Android Studio中，不会自动生成，需要我们手动"Make Projct"或者"Sync Project With Gradle Files"，才会在build/generated/source/aidl/debug/目录下生成AIDL对应的AIDL文件。**如果没有IDE，我们也可以通过sdk build-tools内的aidl.exe程序在dos环境下利用aidl文件生成对应的java文件，命令为`aidl file_name.aidl`，其实，我们也还可以不用编写aidl文件来生成对应的java文件，可以自己手写这个java文件**。下图为Android Studio为我们生成的文件:

![](http://i.imgur.com/K8nOCGS.png)

点进度看IMathAidlInterface.java里面的内容:

![](http://i.imgur.com/kdL5sxn.png)

可以看到IMathAidlInterface是一个接口，继承自android.os.IInterface接口，翻到最下面，就可以看到我们刚才在AIDL文件中定义的加法接口:

![](http://i.imgur.com/SDTcRF2.png)

##### 3、创建远程服务，供客户端远程调用

这里创建一个名为MathService的服务，这里刚才生成的java文件就派上用场了，定义一个IMathAidlInterface接口的实现类mStub，重写add方法，在Service的onBind方法中，return这个mStub，顺便在add方法内这里打印一下Log，方便等会测试。

![](http://i.imgur.com/TUfw8KB.png)

然后就是注册我们的Service服务了：

![](http://i.imgur.com/tx8NZka.png)

注意这里的`android:exported="true"`不能少，否则等会客户端远程绑定服务时会出错，提示不允许绑定服务。

至此服务端创建完毕，下面创建客户端。

### 客户端

##### 1、创建一个名为AIDLClient的project作为客户端
##### 2、讲服务端project中的的AIDL文件夹拷贝到客户端project内(注意aidl文件夹的层级，和java文件夹在同一级)

![](http://i.imgur.com/P4puyfP.png)

然后手动"Make Projct"或者"Sync Project With Gradle Files"，也使其生成对应的IMathAidlInterface.java文件

##### 3、客户端远程绑定服务端的Service

![](http://i.imgur.com/2Fr6EIr.png)

这里我们在客户端的onCreate方法内绑定服务，使客户端启动的时候就立即绑定。然后再onServiceConnected方法内，接收远程服务返回给我们的IMathAidlInterface接口实现对象，有了这个对象，我们就可以调用远程服务的方法了。

![](http://i.imgur.com/hZdXk4w.png)

这里`calculate`为按钮的点击事件，计算5加6的和，toast弹出显示，最后，不要忘记在客户端的onDestroy方法内解绑ServiceConnection对象。

至此，客户端编写完毕，然后先部署服务端，再部署客户端运行查看效果：

![](http://i.imgur.com/vLrpJkl.jpg)

显示的结果为，此时远程服务的add方法也调用，因为从打印的log可以看得出来：

![](http://i.imgur.com/iuTB7pa.png)

至此，一个简单的AIDL Demo远程调用示例演示完毕。

---
## 二、传递自定义类型

传递自定义类型时，需要自定义的类需要序列化，比如实现Parcelable接口，这里我们将上面的例子改成向服务端一个List集合中不断的添加Person对象，然后服务端将集合返回给客户端

首选在服务端定义一个Person类如下：

``` java
public class Person implements Parcelable {

    private String name;
    private String age;

    protected Person(Parcel in) {
        name = in.readString();
        age = in.readString();
    }

    public static final Creator<Person> CREATOR = new Creator<Person>() {
        @Override
        public Person createFromParcel(Parcel in) {
            return new Person(in);
        }

        @Override
        public Person[] newArray(int size) {
            return new Person[size];
        }
    };

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public String getAge() {
        return age;
    }

    public void setAge(String age) {
        this.age = age;
    }

    @Override
    public String toString() {
        return "Person{" +
                "name='" + name + '\'' +
                ", age='" + age + '\'' +
                '}';
    }

    @Override
    public int describeContents() {
        return 0;
    }

    @Override
    public void writeToParcel(Parcel parcel, int i) {
        parcel.writeString(name);
        parcel.writeString(age);
    }
}
```

然后将服务端的aidl文件修改如下所示：

![](http://i.imgur.com/AqNbs69.png)

这时候，我们手动编译下，使其产生对应的java文件，这时候会出错，因为Person虽然实现了序列化，但是这里Person类在aidl中识别不了，我们需要在aidl目录内同时新建一个Person.aidl文件用来标识：

![](http://i.imgur.com/N9yDmDX.png)

这样还不行，还需要在IMathAidlInterface.aidl文件内import这个Person，并且在参数内指定in,out,or inout，这里指定in，如下图所示：

![](http://i.imgur.com/ILI90o3.png)

这个时候再编译就没有问题了，然后把服务端的MathService调整修改一下，如下图所示：

![](http://i.imgur.com/2Glh7EK.png)

服务端修改完毕，接着修改客户端。

首先把服务端的aidl文件夹copy覆盖到客户端，同时也要把客户端java文件夹内刚才实现Parcelable的Person类也要copy到客户端java文件夹内,注意需要保持Person在服务端的包名一致，所以在客户端需要手动创建一个和服务端一样的包名并把Person类放进去，然后把按钮点击事件内的代码删掉，然后编译客户端，应该不会出现什么问题。

![](http://i.imgur.com/xK35Wvc.png)

然后编写客户端的添加逻辑，并打印服务端返回的集合，然后先部署服务端，再部署客户端并运行查看效果：

![](http://i.imgur.com/oFTNhBY.png)


从log来看点击了三次按钮，效果和期望的一样，每次添加一个Person，然后并返回。

至此，成功的演示了利用AIDL实现自定义类型Person的跨进程传输。

下篇文章分析AIDL的原理实现。

---





