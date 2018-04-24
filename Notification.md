# Notification

## 类别

### SimpleNotification
通知样式：
![](https://www.github.com/wslaimin/blog/raw/master/pics/simple_notification.png)
### ProgressNotification
通知样式：
![](https://www.github.com/wslaimin/blog/raw/master/pics/progress_notification.png)
### BigTextNotification
通知收起时样式：
![](https://www.github.com/wslaimin/blog/raw/master/pics/big_text_notification_collapsed.png)
通知展开时样式：
![](https://www.github.com/wslaimin/blog/raw/master/pics/big_text_notification_expanded.png)
### InBoxNotification
通知收起时样式：
![](https://www.github.com/wslaimin/blog/raw/master/pics/in_box_notification_collapsed.png)
通知展开时样式：
![](https://www.github.com/wslaimin/blog/raw/master/pics/in_box_notification_expanded.png)
### BigPictureNotification
通知收起时样式：
![](https://www.github.com/wslaimin/blog/raw/master/pics/big_picture_collapsed.png)
通知展开时样式：
![](https://www.github.com/wslaimin/blog/raw/master/pics/big_picture_expanded.png)
### HangUpNotification
通知样式：
![](https://www.github.com/wslaimin/blog/raw/master/pics/hang_up_notification.png)
### MediaNotification
通知样式：
![](https://www.github.com/wslaimin/blog/raw/master/pics/media_notification.png)
### TemplateNotification
通知样式：
![](https://www.github.com/wslaimin/blog/raw/master/pics/template_notification.png)
##设置通知声音、震动

```java
//设置声音
Builder builder=new NotificationCompat.Builder(context);
......
Uri uri=Uri.parse(ContentResolver.SCHEME_ANDROID_RESOURCE+"://"+context.getPackageName()+"/"+R.raw.song);
builder.setSound(uri);
```

```java
//设置震动
//第一个参数表示震动延迟时间
//第二个参数表示震动持续时间
//第三个参数表示震动后休眠时间
//以此类推
long[] pattern={0,500,500,500,500};
builder.setVibrate(pattern);
```

## 启动Activity保留导航
<dl>
    <dt>
        常规 Activity
    </dt>
    <dd>
        您要启动的
<code><a href="https://developer.android.google.cn/reference/android/app/Activity.html">Activity</a></code> 是应用的正常工作流的一部分。在这种情况下，请设置 <code><a href="https://developer.android.google.cn/reference/android/app/PendingIntent.html">PendingIntent</a></code>
以启动全新任务并为
<code><a href="https://developer.android.google.cn/reference/android/app/PendingIntent.html">PendingIntent</a></code>提供返回栈，这将重现应用的正常“返回”行为。 <i> </i>
        <p>
            Gmail 应用中的通知演示了这一点。点击一封电子邮件消息的通知时，您将看到消息具体内容。
触摸<b>返回</b>将使您从
Gmail 回退到主屏幕，就好像您是从主屏幕（而不是通知）进入
Gmail 一样。
        </p>
        <p>
            无论您触摸通知时处于哪个应用，都会发生这种情况。
例如，如果您在
Gmail 中撰写消息时点击了一封电子邮件的通知，则会立即转到该电子邮件。  <i> </i>
            触摸“返回”会依次转到收件箱和主屏幕，而不是转到您在撰写的邮件。

        </p>
    </dd>
    <dt>
        特殊 Activity
    </dt>
    <dd>
        仅当从通知中启动时，用户才会看到此 <code><a href="https://developer.android.google.cn/reference/android/app/Activity.html">Activity</a></code>。
        从某种意义上说，<code><a href="https://developer.android.google.cn/reference/android/app/Activity.html">Activity</a></code>
是通过提供很难显示在通知本身中的信息来扩展通知。对于这种情况，请将
<code><a href="https://developer.android.google.cn/reference/android/app/PendingIntent.html">PendingIntent</a></code> 设置为在全新任务中启动。但是，由于启动的
<code><a href="https://developer.android.google.cn/reference/android/app/Activity.html">Activity</a></code>
不是应用 Activity 流程的一部分，因此无需创建返回栈。点击“返回”仍会将用户带到主屏幕。<i></i>

    </dd>
</dl>
###设置常规Activity PendingIntent
1.在文件清单定义应用的Activity层次结构

```xml
<activity
    android:name=".MainActivity"
    android:label="@string/app_name" >
    <intent-filter>
        <action android:name="android.intent.action.MAIN" />
        <category android:name="android.intent.category.LAUNCHER" />
    </intent-filter>
</activity>
<activity
    android:name=".ResultActivity"
    android:parentActivityName=".MainActivity">
    <meta-data
        android:name="android.support.PARENT_ACTIVITY"
        android:value=".MainActivity"/>
</activity>
```

`<meta-data>`元素里指定的父Activity的方式，是支持Android4.0.3以及更低版本。
android:parentActivityName属性则支持Android4.1以及更高版本。

2.创建返回栈

```java
Intent resultIntent = new Intent(this, ResultActivity.class);
TaskStackBuilder stackBuilder = TaskStackBuilder.create(this);
// Adds the back stack
stackBuilder.addParentStack(ResultActivity.class);
// Adds the Intent to the top of the stack
stackBuilder.addNextIntent(resultIntent);
// Gets a PendingIntent containing the entire back stack
PendingIntent resultPendingIntent =
        stackBuilder.getPendingIntent(0, PendingIntent.FLAG_UPDATE_CURRENT);
...
NotificationCompat.Builder builder = new NotificationCompat.Builder(this);
builder.setContentIntent(resultPendingIntent);
NotificationManager mNotificationManager =
    (NotificationManager) getSystemService(Context.NOTIFICATION_SERVICE);
mNotificationManager.notify(id, builder.build());
```

###设置特殊Activity PendinIntent
1.在文件清单中，添加android:taskAffinity属性。<br>
与在代码中设置的 FLAG_ACTIVITY_NEW_TASK 标志相结合，这可确保此 Activity 不会进入应用的默认任务。任何具有应用默认关联的现有任务均不受影响。<br>
2.添加android:excludeFromRecents="true"。<br>
将新任务从“最新动态”中排除，这样用户就不会在无意中导航回它。<br>

文件清单如下:

```xml
<activity
    android:name=".ResultActivity"
...
    //android:launchMode设置为singleTask非必须
    android:launchMode="singleTask"
    android:taskAffinity=""
    android:excludeFromRecents="true">
</activity>
...
```

3.构建并发出通知
通过使用 FLAG_ACTIVITY_NEW_TASK 和 FLAG_ACTIVITY_CLEAR_TASK 标志调用 setFlags()，将 Activity 设置为在新的空任务中启动。

代码如下：

```java
// Instantiate a Builder object.
NotificationCompat.Builder builder = new NotificationCompat.Builder(this);
// Creates an Intent for the Activity
Intent notifyIntent =
        new Intent(this, ResultActivity.class);
// Sets the Activity to start in a new, empty task
notifyIntent.setFlags(Intent.FLAG_ACTIVITY_NEW_TASK
                        | Intent.FLAG_ACTIVITY_CLEAR_TASK);
// Creates the PendingIntent
PendingIntent notifyPendingIntent =
        PendingIntent.getActivity(
        this,
        0,
        notifyIntent,
        PendingIntent.FLAG_UPDATE_CURRENT
);

// Puts the PendingIntent into the notification builder
builder.setContentIntent(notifyPendingIntent);
// Notifications are issued by sending them to the
// NotificationManager system service.
NotificationManager mNotificationManager =
    (NotificationManager) getSystemService(Context.NOTIFICATION_SERVICE);
// Builds an anonymous Notification object from the builder, and
// passes it to the NotificationManager
mNotificationManager.notify(id, builder.build());
```