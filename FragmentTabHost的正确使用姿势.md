#FragmentTabHost的正确使用姿势
##使用方法
1.定义布局文件

```xml
<?xml version="1.0" encoding="utf-8"?>
<android.support.constraint.ConstraintLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context="lm.com.experience.tabhost.TabHostActivity">

    <android.support.v4.app.FragmentTabHost
        android:id="@android:id/tabhost"
        android:layout_width="match_parent"
        android:layout_height="match_parent">
        <LinearLayout
            android:layout_width="match_parent"
            android:layout_height="match_parent"
            android:orientation="vertical">
            
            <FrameLayout
                android:id="@android:id/tabcontent"
                android:layout_width="match_parent"
                android:layout_height="0dp"
                android:layout_weight="1"/>
            <TabWidget
                android:id="@android:id/tabs"
                android:layout_width="match_parent"
                android:layout_height="wrap_content"/>
        </LinearLayout>
    </android.support.v4.app.FragmentTabHost>

</android.support.constraint.ConstraintLayout>
```

2.调用setup()方法设置显示内容的布局id，设置tab

```java
mTabHost = (FragmentTabHost) findViewById(android.R.id.tabhost);
        mTabHost.setup(this, getSupportFragmentManager(), android.R.id.tabcontent);
        mTabHost.addTab(mTabHost.newTabSpec("tab1").setIndicator("tab1"), Fragment1.class, null);
        mTabHost.addTab(mTabHost.newTabSpec("tab2").setIndicator("tab2"), Fragment2.class, null);
```

##setup流程分析
注意布局中FrameLayout的id为@android:id/tabcontent,TabWidget的id为@android:id/tabs。前者决定了内容的填充布局(只有布局里TabWidget的id为@android.R.id.tabs有效)，后者决定了FragmentTabHost容器里的层级关系。来靠近源码一点。

FragmentTabHost的setup()方法：

```java
public void setup(Context context, FragmentManager manager, int containerId) {
    ensureHierarchy(context);  
    super.setup();
    mContext = context;
    mFragmentManager = manager;
    mContainerId = containerId;
    ensureContent();
    mRealTabContent.setId(containerId);
    // We must have an ID to be able to save/restore our state.  If
    // the owner hasn't set one at this point, we will set it ourselves.
    if (getId() == View.NO_ID) {
        setId(android.R.id.tabhost);
    }
}
```
上面的ensureHierarchy()和ensureContent()方法是我们需要关注的方法。
ensureHierarchy()方法保证容器里显示内容和tab的层级关系：

```java
 private void ensureHierarchy(Context context) {
    // If owner hasn't made its own view hierarchy, then as a convenience
    // we will construct a standard one here.
    if (findViewById(android.R.id.tabs) == null) {
        LinearLayout ll = new LinearLayout(context);        
        ll.setOrientation(LinearLayout.VERTICAL);
        addView(ll, new FrameLayout.LayoutParams(
                ViewGroup.LayoutParams.MATCH_PARENT,
                ViewGroup.LayoutParams.MATCH_PARENT));

        TabWidget tw = new TabWidget(context);
        tw.setId(android.R.id.tabs);
        tw.setOrientation(TabWidget.HORIZONTAL);            
        ll.addView(tw, new LinearLayout.LayoutParams(
                ViewGroup.LayoutParams.MATCH_PARENT,
                ViewGroup.LayoutParams.WRAP_CONTENT, 0));

        FrameLayout fl = new FrameLayout(context);
        fl.setId(android.R.id.tabcontent);
        ll.addView(fl, new LinearLayout.LayoutParams(0, 0, 0));             
        mRealTabContent = fl = new FrameLayout(context);
        mRealTabContent.setId(mContainerId);
        ll.addView(fl, new LinearLayout.LayoutParams(
                LinearLayout.LayoutParams.MATCH_PARENT, 0, 1));
    }
}
```

确定层级关系的逻辑是，id为android.R.id.tabs的TabWidget不存在，则创建id为android.R.id.tabs的TabWidget并且创建id为android.R.id.tabcontent的FrameLayout来作为内容显示的布局。

然后ensurecontent()保证内容显示的布局：

```java
private void ensureContent() {
    if (mRealTabContent == null) {
        mRealTabContent = (FrameLayout)findViewById(mContainerId);          
        if (mRealTabContent == null) {
            throw new IllegalStateException(
                    "No tab content FrameLayout found for id " + mContainerId);
        }
    }
}
```

当mRealTabContent为空时，也就是没有自动创建内容显示布局时，设置为指定id的内容显示布局。

FragmentTabHost的setup()方法的流程：

![](https://www.github.com/wslaimin/blog/raw/master/pics/FragmentTabHost.png)





 
