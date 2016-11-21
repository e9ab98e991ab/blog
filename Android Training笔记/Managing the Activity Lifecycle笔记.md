[TOC]
#前台和后台
前台：Activity正在屏幕上显示，能够与用户进行交互，此时称Activity在前台。
后台：Activity切换到别的应用，或切换到另一Activity，此时称Activity进入后台。

#Activity生命周期
![](https://github.com/wslaimin/blog/raw/master/pics/lifecycle.png)

>注意：只有三种状态是可以长时间保持的：resumed、paused、stopped，其他状态都会很快进入下一个状态。

Java程序的入口是main()，App声明一个main Activity作为入口，点击图标就会启动该Activity。声明如下：

```xml
<activity android:name=".MainActivity" android:label="@string/app_name">
    <intent-filter>
        <action android:name="android.intent.action.MAIN" />
        <category android:name="android.intent.category.LAUNCHER" />
    </intent-filter>
</activity>
```

>注意：没有声明main Activity在桌面不会有app图标。

Activity处于onStopped状态，在极端情况下，系统可能会杀死app进程，不会调用activity的onDestroy()方法，因此要在onStop()中释放资源防止内存泄漏。

>注意：onPause()不适合耗时操作，onStop()可以进行CPU密集型、更耗时的资源释放操作。

当系统配置发生改变时，比如屏幕旋转，有可能在横屏状态需要加载不同的layout，所以系统会销毁现在Activity然后重建。

