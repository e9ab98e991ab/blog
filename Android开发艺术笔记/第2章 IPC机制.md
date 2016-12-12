#第2章 IPC机制
##Android IPC简介
线程是CPU调度的最小单元，同时线程是一种有限的系统资源。而进程一般指一个执行单元，在PC和移动设备上指一个程序或者一个应用。

在Android中最有特色的进程间的通信方式就是Binder。Android对单个应用所使用的最大内存做了限制，早期的一些版本可能是16MB，不同设备有不同的大小。

##Android中的多进程模式
通过给四大组件指定android:process属性，可以很轻易地开启多进程模式。

在Android中多进程是指一个应用中存在多个进程的情况。默认进程的进程名是包名。可以用命令查看进程：adb shell ps或者adb shell ps | grep com.ryg.chapter(包名)。

android:process属性分别为":remote"和"com.ryg.chapter"的区别？首先，":"的含义是指要在当前的进程名前面附加上当前的包名，如"com.ryg,chapter:remote"，后者是一种完整的命名方式，不会附加包名信息；其次，进程名以":"开头的进程属于当前应用的私有进程，其他应用的组件不可以和它跑在同一个进程中，而进程名不以":"开头的进程属于全局进程，其他应用通过ShareUID方式可以和它跑在同一个进程中。

Android为每一个应用分配了一个独立的虚拟机，或者说为每个进程都分配一个独立的虚拟机，不同的虚拟机在内存分配上有不同的虚拟地址空间，这就导致在不同的虚拟机中访问同一个类的对象会产生多份副本。

一般来说，使用多进程会造成如下几方面的问题：
(1)静态成员和单例模式完全失效。
(2)线程同步机制完全失效。
(3)SharedPreference的可靠性下降。
(4)Application会多次创建。

(1)和(2)是由于每个进程有不同内存空间不同对象造成，(3)是SharedPreference不支持两个进程同时去执行写操作，(4)是不同进程的组件会拥有独立的虚拟机、Application以及内存空间。
##IPC基础概念介绍
想让一个对象实现序列化，只需要这个类实现Serializable接口并声明一个serialVersionUID即可，serialVersionUID也并不是必需的，但是这将会对反序列化过程产生影响。

```java
//序列化过程
User user=new User(0,"jake",true);
ObjectOutputStream out=new ObjectOutputStream(new FileOutputStream("cache.txt"));
out.writeObject(user);
out.close();

//反序列化过程
ObjectInputStream in=new ObjectInputStream(new FileInputStream("cache.txt"));
User newUser=(User)in.readObject();
in.close();
```

原则上序列化后的数据中的serialVersionUID只有和当前类的serialVersionUID相同才能够正常地被反序列化。不指定serialVersionUID的值，当类有所改变，系统会重新计算当前类的hash值并把它赋值给serialVersionUID；反之，类发生改变，反序列化仍然能够成功，最大限度地恢复数据。
>注意：静态成员变量属于类不属于对象，所以不会参与序列化过程；用transient关键字标记的成员变量不参与序列化过程。

Android特有的序列化接口Parcelable:序列化功能由writeToParcel方法完成，反序列化功能由CREATOR来完成，几乎所有情况下describeContents方法都应该返回0，仅当当前对象中存在文件描述符时，此方法返回1。对象中成员变量也是可序列化对象时，反序列化过程需要传递当前线程的上下文类加载器，否则会报无法找到类的错误。如下：

```java
book=in.readParcelable(Thread.currentThread().getContextClassLoader());
```

从IPC角度来说，Binder是Android中的一种跨进程通信方式。Android开发中，Binder主要用在Service中。

使用Binder例子：新建Java包com.ryg.chapter_2.aidl，然后新建三个文件Book.java、Book.aidl和IBookManager.aidl。Book.java是一个表示图书信息的类，它实现了Parcelable接口。Book.aidl是Book类在AIDL中的声明(要和相应的Java类同名)。IBookManager.aidl是定义的一个接口。

生成的内部类Stub是一个Binder类，当客户端和服务端都位于同一个进程时，方法调用不会走跨进程的transact过程，而当两者位于不同进程时，方法调用需要走transact过程，这个逻辑由Stub的内部代理类Proxy来完成。Proxy中完成序列化过程，onTransact完成反序列化过程。

针对Stub类和Stub的内部代理类Proxy介绍：
DESCRIPTOR
Binder的唯一标识，一般用当前Binder的类名表示，如"com.ryg.chapter_2.aidl.IBookManager"。

asInterface(android.os.IBinder obj)
用于将服务端的Binder对象转换成客户端所需要的AIDL接口类型的对象，这种转换过程是区分进程的，如果客户端和服务端位于同一进程，那么此方法返回的就是服务端的Stub对象本身，否则返回的是系统封装后的Stub.Proxy对象。

asBinder
此方法返回当前Binder对象。

onTransact
这个方法运行在服务端的Binder线程池中。方法原型为public Boolean onTransact(int code,android.os.Parcel data,android.os.Parcel reply,int flags)。code可以确定客户端所请求的目标方法是什么，从data中取出目标方法所需的参数，目标方法执行完后，就向reply中写入返回值。如果此方法返回false，那么客户端的请求会失败。

>注意：首先，当客户端发起远程请求时，由于当前线程会被挂起直至服务端进程返回数据，所以如果一个远程方法是很耗时的，那么不能在UI线程中发起此远程请求；其次，由于服务端的Binder方法运行在Binder线程池中，所以Binder方法不管是否耗时都应该采用同步的方式去实现。

Binder的工作机制图：
![](https://www.github.com/wslaimin/blog/raw/master/binder_jz.jpg)

Binder运行在服务端进程。Binder中提供了两个配对的方法linkToDeath和unlinkToDeath，通过linkToDeath可以给Binder设置一个死亡代理，当Binder死亡时，会收到通知，这个时候就可以重新发起连接请求从而恢复连接。

给Binder设置死亡回调：

```java
private IBinder.DeathRecipient mDeathRecipient=new IBinder.DeathRcipient(){
    @Override
    public void binderDied(){
        if(mBookManager==nuol)
            return;
        mBookManager.asBinder().unlinkToDeath(mDeathRecipient,0);
        mBookManager=null;
        //TODO:这里重新绑定远程Service
    }
}
```

在客户端绑定远程服务成功后，给Binder设置死亡代理：

```java
mService=IMessageBoxManager.Stub.asInterface(binder);
binder.linkToDeath(mDeathRecipient,0);
```

其中linkToDeath的第二个参数是个标记位，直接设为0即可。

##Android中的IPC方式
Android中的IPC方式：可以采用Binder方式来跨进程通信；ContentProvider天生就是支持跨进程访问的；Socket也可以实现IPC。

四大组件中的三大组件(Activity、Service、Receiver)都支持在Intent中传递Bundle数据，由于Bundle实现了Parcelable接口，所以它可以方便地再不同的进程间传输。

文件共享方式适合在对数据同步要求不高的进程之间进行通信。SharedPreferences在内存中会有一份文件的缓存，在多进程模式下，系统对它的读/写就变得不可靠，当面对高并发的读/写访问，SharedPreferences有很大几率会丢失数据，因此，不建议在进程间通信中使用SharedPreferences。

###使用Messenger
Messenger的底层实现是AIDL。它一次处理一个请求，无需考虑线程同步问题。Message中的字段object在同一个进程中是很实用的，但是在进程间通信时，也仅仅是系统提供的实现了Parcelable接口的对象才能通过它来传输，我们自定义的Parcelable对象是无法通过object字段来传输的。所幸我们还有Bundle字段可以使用。Messager工作原理：
![](https://github.com/wslaimin/blog/raw/master/pics/messenger.jpg)

Messenger是以串行的方式处理客户端发来的消息，如果大量的消息同时发送到服务器，服务端仍然只能一个个处理，如果有大量的并发请求，那么用Messenger就不太合适了。同时，Messenger的作用主要是为了传递消息。处理多并发可以使用AIDL来实现跨进程的方法调用。

###使用AIDL
AIDL文件支持的数据类型：
1、基本数据类型(int、long、char、boolean、double等以及基本数据类型数组)
2、String和CarSequence
3、List:只支持ArrayList，里面每个元素都必须能够被AIDL支持
4、Map：只支持HashMap，里面每个元素都必须被AIDL支持，包括key和value
5、Parcelable:所有实现了Parcelable接口的对象
6、AIDL：所有的AIDL接口本身也可以在AIDL文件中使用

自定义的Parcelabe对象和AIDL对象必须要显式import进来，不管它们是否和当前的AIDL文件位于同一个包内。如果AIDL文件中用到了自定义的Parcelable对象，那么必须新建一个和它同名的AIDL文件，并在其中声明它为Parcelable类型。

AIDL中除了基本数据类型，其他类型的参数必须标上方向：in、out或者inout，in表示输入类型，out表示输出类型参数，inout表示输入输出参数。AIDL接口中只支持方法，不支持声明静态常量，这一点区别于传统的接口。

AIDL的包结构在服务端和客户端要保持一致，否则运行会出错，这是因为客户端需要发序列化服务端中和AIDL接口相关的所有类，如果类的完整路径不一样的话，是无法成功反序列化。

AIDL中所支持的是抽象的List，而List只是一个接口，因此虽然服务端返回的是CopyOnWriteArrayList，但是在Binder中会按照List的规范去访问数据并最终形成一个新的ArrayList传递给客户端。

对象是不能跨进程直接传输的，对象的跨进程传输本质上都是反序列化的过程，这就是为什么AIDL中的自定义对象都必须要实现Parcelable接口的原因。

RemoteCallbackList是系统专门提供的用于删除跨进程listener的接口。当客户端进程终止后，它能够自动移除客户端所注册的listener。

虽然说多次跨进程传输客户端的同一个对象会在服务端生成不同的对象，但是这些新生成的对象有一个共同点，那就是它们底层的Binder对象时同一个。

使用RemoteCallbackList,有一点需要注意，我们无法像操作List一样去操作它，尽管它的名字也带个List，但是它并不是一个List。

```java
final inte N=mListenerList.beginBroadcast();
for(int i=0;i<N;i++){
    IOnNewBookArrivedListener l=mListenerList.getBroadcastItem(i);
    if(l!=null){
        //TODO handle l
    }
}
mListenerList.finishBroadcast();
```

当远程服务端需要调用客户端的listener中的方法时，被调用的方法也运行在Binder线程池中，只不过是客户端的线程池。所以，同样不可以在服务端中调用客户端的耗时方法。

Binder意外死亡的，重新连接服务有两种方法。第一种方法是给Binder设置DeathRecipient监听；第二种方法是在onServiceDisconnected中重连远程服务。区别在于：onServiceDisconnected在客户端的UI线程中被回调，而binderDied在客户端的Binder线程池中被回调。

可以在服务端的onTransact方法中进行权限验证。

###使用ContentProvider
ContentProvider是Android中提供的专门用于不同应用间进行数据共享的方式，ContentProvider的底层实现同样也是Binder。

创建一个自定义的ContentProvider需要继承ContentProvider类并实现六个抽象方法：onCreate、query、update、insert、delete和getType。getType用来返回一个Uri请求所对应的MIME类型(媒体类型)。除了onCreate由系统调用并运行在主线程里，其他五个方法均由外界回调并运行在Binder线程池中。

ContentProvider需要在manifest文件注册，android:authorities是ContentProvider的唯一标识，通过这个属性外部应用就可以访问定义的ContentProvider，因此，android:authorities必须是唯一的。

ContentProvider通过Uri来区分外界要访问的数据集合。根据请求的Uri来得到Uri Code，有了Uri Code就可以知道外界想要访问哪个表。

update、insert、delete方法会引起数据源的改变，需要通过ContentResolver的notifyChange方法来通知外界当前ContentProvider中的数据已经发生改变。要观察一个ContentProvider中数据改变的情况，可以通过ContentResolber的registerContentObserver方法来注册观察者，通过unRegisterContentObserver方法来接触观察者。

query、update、insert、delete四大方法是存在多线程并发访问的，因此方法内部要做好线程同步。SQLiteDatabase内部对数据库的操作是有同步处理的，但是如果通过多个SQLiteDatabase对象来操作数据库就无法保证线程同步，因为SQLiteDatabase对象之间无法进行线程同步。

IPC总结
![](https://github.com/wslaimin/blog/raw/master/pics/ipc.jpg)
  



