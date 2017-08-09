#android:clipChildren属性
##属性介绍
android:clipChildren用来设置是否剪切children。
##使用场景
通常会看到中间的tab会更大，甚至超出parent的限制，如图：
![](https://www.github.com/wslaimin/blog/raw/master/pics/clipChildren.png)

##属性使用方法
android:clipChildren="false"表示不剪切children，但是children大小不能超过parent的大小，所以如果children的直接parent大小不够容纳children的大小，那么要一直向根布局设置android:clipChildren="false"，知道有parent能够容纳children的大小。

activity布局如下：
activity_main.xml

```xml
<?xml version="1.0" encoding="utf-8"?>
<android.support.v4.app.FragmentTabHost xmlns:android="http://schemas.android.com/apk/res/android"
    android:id="@android:id/tabhost"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:orientation="vertical"
        android:clipChildren="false">
        <FrameLayout
            android:id="@android:id/tabcontent"
            android:layout_width="match_parent"
            android:layout_height="0dp"
            android:layout_weight="1"
            android:background="#000000" />
        <TabWidget
            android:id="@android:id/tabs"
            android:layout_width="match_parent"
            android:layout_height="45dp"
            android:clipChildren="false"/>
    </LinearLayout>
</android.support.v4.app.FragmentTabHost>
```

中间tab的布局如下：
tab_big_item_view.xml

```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_height="wrap_content"
    android:layout_width="wrap_content"
    android:orientation="vertical"
    android:clipChildren="false"
    android:gravity="center">

    <ImageView
        android:focusable="false"
        android:id="@+id/imageview"
        android:layout_height="60dp"
        android:layout_width="60dp"
        android:scaleType="fitXY"/>

    <TextView
        android:id="@+id/textview"
        android:layout_height="wrap_content"
        android:layout_width="wrap_content"
        android:text=""
        android:textColor="#ffffff"
        android:textSize="10sp"/>

</LinearLayout>
```

从上面可以看出TabWidget的高度为45dp，但是tab的高度为45dp，但是由于设置android:clipChildren="false"，tab的显示高度为60dp。

ps：可以通过parent的android:gravity或自身的android:layout_gravity属性来控制超出部分的显示。

<a href="https://github.com/wslaimin/FragmentTabHost.git">完整例子</a>