#第5章 理解RemoteViews
RemoteViews表示的是一个View结构，它可以在其他进程中显示，提供了一组基础的操作用于跨进程更新它的界面。
##5.1 RemoteViews的应用
RemoteViews在实际开发中，主要用在通知栏和桌面小部件的开发过程中。通知栏和桌面小部件都运行在系统的SystemServer进程。

###RemoteViews在通知栏上的应用
自定义通知栏示例：

```java
NotificationCompat.Builder builder1=new NotificationCompat.Builder(this)
    .setTicker("this is a test")
    .setSmallIcon(R.drawable.ic_launcher)
    .setAutoCancel(true)
    .setTicker("hello world")
    .setWhen(System.currentTimeMillis());
    Intent intent1=new Intent(this,TestNotification.class);
    PendingIntent pendingIntent1=PendingIntent.getActivity(this,0,intent1,PendingIntent.FLAG_UPDATE_CURRENT);
    RemoteViews remoteViews=new RemoteViews(getPackageName(),R.layout.notification);
    remoteViews.setTextViewText(R.id.text,"test");
    remoteViews.setTextColor(R.id.text, Color.BLACK);
    remoteViews.setImageViewResource(R.id.image,R.drawable.ic_launcher);
    PendingIntent pendingIntent2=PendingIntent.getActivity(this,0,new Intent(this, TestNotification.class),PendingIntent.FLAG_UPDATE_CURRENT);
    remoteViews.setOnClickPendingIntent(R.id.image,pendingIntent2);
    builder1.setContent(remoteViews)
            .setContentIntent(pendingIntent1);
    NotificationManager manager1=(NotificationManager)getSystemService(Context.NOTIFICATION_SERVICE);
    manager1.notify(2,builder1.build());
```

###RetoteViews在桌面小部件上的应用
AppWidgetProvider是Android中提供的用于实现桌面小部件的类，其本质是一个广播，即BroadcastReceiver。
 - 定义小部件界面

```xml
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:orientation="vertical"
    android:layout_width="match_parent"
    android:layout_height="match_parent">
    <Button
        android:id="@+id/button_system_notification"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:onClick="onClick"/>

    <Button
        android:id="@+id/button_custom_notification"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:onClick="onClick"/>
</LinearLayout>
```

 - 定义小部件配置信息
 
 在res/xml下新建appwidget_provider_info，名称随意选择：

```xml
<appwidget-provider xmlns:android="http://schemas.android.com/apk/res/android"
    android:initialLayout="@layout/notification"
    android:minHeight="50dp"
    android:minWidth="200dp"
    android:updatePeriodMillis="6000">
</appwidget-provider>
```

updatePeriodMillis表示小工具的自动更新周期，单位为毫秒。实际上最小更新周期为30min，不能小于这个值。

 - 继承AppWidgetProvider

```java
public class TestAppWidgetProvider extends AppWidgetProvider{
    public static final String CLICK_ACTION="com.example.lm.action.CLICK";

    @Override
    public void onReceive(final Context context, Intent intent) {
        super.onReceive(context, intent);
        System.out.println("received");

        if(intent.getAction().equals(CLICK_ACTION)){
            Toast.makeText(context,"click it",Toast.LENGTH_LONG).show();
            final Bitmap bitmap = BitmapFactory.decodeResource(context.getResources(),
                    R.drawable.ic_launcher);
            final AppWidgetManager appWidgetManager=AppWidgetManager.getInstance(context);
            new Thread(new Runnable() {
                @Override
                public void run() {
                    for (int i = 0; i <= 36; i++) {
                        Bitmap rotateBitmap = rotateBitmap(bitmap, (i * 10) % 360);
                        RemoteViews remoteViews=new RemoteViews(context.getPackageName(),
                                R.layout.notification);
                        remoteViews.setImageViewBitmap(R.id.image,rotateBitmap);
                        Intent intent=new Intent();
                        intent.setAction(CLICK_ACTION);
                        PendingIntent pendingIntent=PendingIntent.getBroadcast(context,0,intent,0);
                        remoteViews.setOnClickPendingIntent(R.id.image,pendingIntent);
                        appWidgetManager.updateAppWidget(new ComponentName(context,TestAppWidgetProvider.class),
                                remoteViews);
                        SystemClock.sleep(30);
                    }
                }
            }).start();
        }
    }

    @Override
    public void onUpdate(Context context, AppWidgetManager appWidgetManager, int[] appWidgetIds) {
        super.onUpdate(context, appWidgetManager, appWidgetIds);
        Bitmap bitmap = BitmapFactory.decodeResource(context.getResources(),R.drawable.ic_launcher);
        RemoteViews remoteViews=new RemoteViews(context.getPackageName(),
                R.layout.notification);
        remoteViews.setImageViewBitmap(R.id.image,bitmap);
        Intent intent=new Intent();
        intent.setAction(CLICK_ACTION);
        PendingIntent pendingIntent=PendingIntent.getBroadcast(context,0,intent,0);
        remoteViews.setOnClickPendingIntent(R.id.image,pendingIntent);
        appWidgetManager.updateAppWidget(appWidgetIds,remoteViews);

        System.out.println("update");
    }

    private Bitmap rotateBitmap(Bitmap bitmap, float degree){
        Matrix matrix=new Matrix();
        matrix.setRotate(degree);
        return Bitmap.createBitmap(bitmap,0,0,bitmap.getWidth(),bitmap.getHeight(),matrix,true);
    }
}
```

上面代码的主要实现是单击桌面小部件图片，更新RemoteViews来使图片旋转。

 - 在AndroidManifest.xml中声明小部件

```xml
<receiver android:name=".remoteview.TestAppWidgetProvider">
    <meta-data
        android:name="android.appwidget.provider"
        android:resource="@xml/appwidet_provider_info"/>
            <intent-filter>
                <action android:name="com.example.lm.action.CLICK"/>
                <action android:name="android.appwidget.action.APPWIDGET_UPDATE"/>
            </intent-filter>
</receiver>
```

上面代码中有两个Action，第一个Action用于识别小部件的单击行为，第二个作为小部件的标识必须存在。

AppWidgetProvider除了最常用的onUpdate方法，还有以下几个方法：

 1. onEnable:当该窗口小部件第一次添加到桌面时调用该方法，可添加多次但只在第一次调用。
 2. onUpdate:小部件被添加时或者每次小部件更新时都会调用一次该方法，小部件的更新时机由updatePeriodMills来指定。
 3. onDelete:每删除一次桌面小部件就调用一次。
 4. onDisable:当最后一个该类型的桌面小部件被删除时调用该方法，注意是最后一个。
 5. onReceive:这是广播的内置方法，用于分发具体的事件给其他方法。
 
##PendingIntent概述
PendingIntent表示一种处于pending状态的意图，而pending状态表示的是一种待定、等待、即将发生的意思。由于RemoteViews运行在远程进程中，因此无法直接像View那样通过setOnClickListener方法来设置单击事件。

PedingIntent支持三种待定意图：启动Activity、启动Service和发送广播。

通过PendingIntent的getActivity(Context context,int requestCode,Intent intent,int flags)可以获得开启Activity的PendingIntent。requestCode表示PendingIntent发送方的请求码，多数情况下设为0即可，另外requestCode会影响到flag的效果。

两个PendingIntent是相同的情况：如果两个PendingIntent它们内部的Intent相同并且requestCode也相同，那么这两个PendingIntent就相同。Intent相同的匹配规则是：如果两个Intent的ComponentName和intent-filter都相同，那么这两个Intent就是相同的。可以参看Intent的filterEquals()方法。
flags参数有一下几种：

 - FLAG_ONE_SHOT
 当前描述的PendingIntent只能被使用一次，然后它就会被自动cancel,如果后续还有相同的PendingIntent，那么它们的send方法就会调用失败。对于通知栏消息来说，如果采用此标记位，那么同类的通知只能使用一次，后续的通知单击后将无法打开。
 - FLAG_NO_CREATE
 当前描述的PendingIntent不会主动创建，如果当前PendingIntent之前不存在，那么getActivity、getService、getBroadcast方法会直接返回null,即获取PendingIntent失败。这个标记位很少使用。
 - FLAG_CANCEL_CURRENT
当前描述的PendingIntent如果已经存在，那么它们都会被cancel，然后系统会创建一个新的PendingIntent。对于通知栏消息来说，那些被cancel的消息单击后将无法打开，只有最新的通知栏消息点击后能够打开。
 - FLAG_UPDATE_CURRENT
当前描述的PendingIntent如果已经存在，那么它们都会被更新，即它们的Intent中的Extras会被替换成最新的。

对于通知栏消息来说，如果notify的第一个参数id是常量，那么多次调用notify只能弹出一个通知，后续的通知会把前面的通知完全替代掉。

##RemoteViews的内部机制
RemoteViews目前并不能支持所有的View类型，它所支持的所有类型如下：

 - Layout
FrameLayout、LinearLayout、RelativeLayout、GridLayout
 - View
AnalogClock、Button、Chronometer、ImageButton、ImageView、ProgressBar、TextView、ViewFlipper、ListView、GridView、StackView、AdapterViewFlipper、ViewStub

上面所述是RemoteViews所支持的所有View类型，RemoteViews不支持它们的子类以及其他View类型，也无法使用自定义View。

RemoteViews的一系列set方法是通过反射来完成的。

RemoteViews实现了parcelable接口，因此它可以跨进程传输。Action代表一个View操作，Action同样实现了Parcelable接口。系统首先将View操作封装到Action对象并将这些对象跨进程传输到远程进程，接着在远程进程中执行Action对象中的具体操作。每调用一次set方法，RemoteViews中就会添加一个对应的Action对象。过程如图：
![](https://www.github.com/wslaimin/blog/raw/master/pics/remoteviews.jpg)

远程进程调用RemoteViews的apply或reApply方法来执行Action。apply和reApply的区别在于：apply会加载布局并更新界面，而reapply则只会更新界面。通知栏和桌面小插件在初始化界面时会调用apply方法，而在后续的更新界面时会调用reapply方法。

setOnClickPendingIntent用于给普通View设置单击事件，但是不能给集合(ListView和StackView)中的View设置单击事件。如果要给ListView和StackView中的Item添加单击事件，则必须将setPendingIntentTemplate和setOnClickFillIntent组合使用才可以。

给ListView的Item设置单击事件的步骤：

 - 用setPendingIntentTemplate设置Intent模板

```java
public class TestListViewClick extends AppWidgetProvider{
    @Override
    public void onReceive(Context context, Intent intent) {
        super.onReceive(context, intent);
    }

    @TargetApi(Build.VERSION_CODES.ICE_CREAM_SANDWICH)
    @Override
    public void onUpdate(Context context, AppWidgetManager appWidgetManager, int[] appWidgetIds) {
        super.onUpdate(context, appWidgetManager, appWidgetIds);
        for(int i=0;i<appWidgetIds.length;i++){
            RemoteViews remoteViews=new RemoteViews(context.getPackageName(),
                    R.layout.widget_listview);
            //RemoteViews Service needed to provide adapter for ListView
            Intent svcIntent = new Intent(context, WidgetService.class);
            //passing app widget id to that RemoteViews Service
            svcIntent.putExtra(AppWidgetManager.EXTRA_APPWIDGET_ID, appWidgetIds[i]);
            //setting a unique Uri to the intent
            //don't know its purpose to me right now
            svcIntent.setData(Uri.parse(svcIntent.toUri(Intent.URI_INTENT_SCHEME)));
            //setting adapter to listview of the widget
            remoteViews.setRemoteAdapter(appWidgetIds[i], R.id.listView,
                    svcIntent);
            //setting an empty view in case of no data
            remoteViews.setEmptyView(R.id.listView, R.id.empty_view);
            PendingIntent pendingIntent=PendingIntent.getActivity(context,0,
                    new Intent(context,TestNotification.class),0);
            remoteViews.setPendingIntentTemplate(R.id.listView,pendingIntent);
            appWidgetManager.updateAppWidget(appWidgetIds[i],remoteViews);
        }
    }
}
```

setRemoteAdapter()方法启动远程RemoteService来填充ListView。

 - 继承RemoteService，实现远程填充ListView
 
 ```java
 public class WidgetService extends RemoteViewsService{
    @Override
    public RemoteViewsFactory onGetViewFactory(Intent intent) {
        return new ListViewFactory(getApplication());
    }
}
 ```
 
 
 - 实现RemoteViewsService.RemoteViewsFactory接口，这个过程和普通继承BaseAdpter差不多

```java
@Override
    public RemoteViews getViewAt(int position) {
        RemoteViews remoteViews=new RemoteViews(mContext.getPackageName(),
                R.layout.notification);
        /*remoteViews.setTextViewText(android.R.id.text1,mData.get(position));
        remoteViews.setTextColor(android.R.id.text1, Color.WHITE);*/
        Intent intent=new Intent();
        remoteViews.setOnClickFillInIntent(R.id.layout,intent);
        return remoteViews;
    }
```

远程Item的RemoteViews调用setOnClickFillInIntnt()方法填充之前的Intent模板，两者结合成最终Intent。
 
 
 
 
 
 
  

