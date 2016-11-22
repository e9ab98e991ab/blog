#RadioGroup调用check()方法onCheckedChanged()调用多次
##情景再现
布局文件activity_main.xml如下：

```xml
<?xml version="1.0" encoding="utf-8"?>
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:id="@+id/activity_main"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:paddingBottom="@dimen/activity_vertical_margin"
    android:paddingLeft="@dimen/activity_horizontal_margin"
    android:paddingRight="@dimen/activity_horizontal_margin"
    android:paddingTop="@dimen/activity_vertical_margin"
    tools:context="example.lm.com.testradiogroup.MainActivity">
    <RadioGroup
        android:id="@+id/group"
        android:layout_width="match_parent"
        android:layout_height="wrap_content">
        <RadioButton
            android:id="@+id/button1"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:text="button1"/>
        <RadioButton
            android:id="@+id/button2"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:text="button2"/>
    </RadioGroup>

    <Button
        android:id="@+id/check"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_below="@id/group"
        android:text="check"/>
</RelativeLayout>
```

MainActivity.java文件如下：

```java
@Override
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.activity_main);
    mGroup=(RadioGroup)findViewById(R.id.group);
    mButton1=(RadioButton)findViewById(R.id.button1);
    mButton2=(RadioButton)findViewById(R.id.button2);
    mGroup.setOnCheckedChangeListener(new RadioGroup.OnCheckedChangeListener() {
        @Override
        public void onCheckedChanged(RadioGroup group, int checkedId) {
            System.out.println(checkedId);
        }
    });

    mCheck=(Button)findViewById(R.id.check);
    mCheck.setOnClickListener(new View.OnClickListener() {
        @Override
        public void onClick(View v) {
            mGroup.check(R.id.button2);
        }
    });

    mButton1.setChecked(true);
}
```

当点击mCheck按钮时，输出结果:

```
11-21 15:37:42.569 8313-8313/example.lm.com.testradiogroup I/System.out: 2131427414
11-21 15:37:42.569 8313-8313/example.lm.com.testradiogroup I/System.out: 2131427415
11-21 15:37:42.569 8313-8313/example.lm.com.testradiogroup I/System.out: 2131427415
```

这说明onCheckedChanged()方法连续调用了3次，可是明明只调用了一次RadioGroup的check()方法。这是怎么回事？这里先说明下，一次是RadioGroup中RadioButton发生改变时调用,一次是由于mButton1由选中状态变为未选中状态时调用，一次是mButton2由未选中状态变为选中状态时调用。

##源码分析
首先，看看RadioGroup的check()方法具体实现：

```java
public void check(@IdRes int id) {
    // don't even bother
    if (id != -1 && (id == mCheckedId)) {
        return;
    }
    if (mCheckedId != -1) {
        setCheckedStateForView(mCheckedId, false);
    }
    if (id != -1) {
        setCheckedStateForView(id, true);
    }

    setCheckedId(id);
}
```

很简单，check()方法里调用了两个方法setCheckedStateForView()和setCheckedId()。先看setCheckedId()方法的实现：

```java
private void setCheckedId(@IdRes int id) {
    mCheckedId = id;
    if (mOnCheckedChangeListener != null) {
        mOnCheckedChangeListener.onCheckedChanged(this, mCheckedId);
    }
}
```

很清楚的看到这里调用了一次onCheckedChanged()方法。接着看setCheckedStateForView()方法的实现：

```java
private void setCheckedStateForView(int viewId, boolean checked) {
    View checkedView = findViewById(viewId);
    if (checkedView != null && checkedView instanceof RadioButton) {
        ((RadioButton) checkedView).setChecked(checked);
    }
}
```

也很简单，只是调用了RadioButton的setChecked()方法：

```java
public void setChecked(boolean checked) {
    if (mChecked != checked) {
        mChecked = checked;
        refreshDrawableState();
        notifyViewAccessibilityStateChangedIfNeeded(
                AccessibilityEvent.CONTENT_CHANGE_TYPE_UNDEFINED);
        // Avoid infinite recursions if setChecked() is called from a listener
        if (mBroadcasting) {
            return;
        }

        mBroadcasting = true;
        if (mOnCheckedChangeListener != null) {
            mOnCheckedChangeListener.onCheckedChanged(this, mChecked);
        }
        if (mOnCheckedChangeWidgetListener != null) {
            mOnCheckedChangeWidgetListener.onCheckedChanged(this, mChecked);
        }

        mBroadcasting = false;            
    }
}
```

这里有调用onCheckedChanged()方法，可是，我们没有给RadioButton调用setOnCheckedChangeListener()方法啊，所以mOnCheckedChangeListener==null，这是怎么回事，其余的两次哪里调用的。如果有调用只能是mOnCheckedChangeWidgetListener.onCheckedChanged()里调用了。

RadioGroup在添加Child的时候会给Child设置CompoundButton.OnCheckedChangeListener：

```java
public void onChildViewAdded(View parent, View child) {
    if (parent == RadioGroup.this && child instanceof RadioButton) {
        int id = child.getId();
        // generates an id if it's missing
        if (id == View.NO_ID) {
            id = View.generateViewId();
            child.setId(id);
        }
        ((RadioButton) child).setOnCheckedChangeWidgetListener(
            mChildOnCheckedChangeListener);
    }

    if (mOnHierarchyChangeListener != null) {
        mOnHierarchyChangeListener.onChildViewAdded(parent, child);
    }
}
```

mChildOnCheckedChangeListener是CheckedStateTracker类的实例，CheckedStateTracker实现：

```java
private class CheckedStateTracker implements CompoundButton.OnCheckedChangeListener {
    public void onCheckedChanged(CompoundButton buttonView, boolean isChecked) {
        // prevents from infinite recursion
        if (mProtectFromCheckedChange) {
            return;
        }
        mProtectFromCheckedChange = true;
        if (mCheckedId != -1) {
            setCheckedStateForView(mCheckedId, false);
        }
        mProtectFromCheckedChange = false;
        int id = buttonView.getId();
        setCheckedId(id);
    }
}
```

终于看到了，这里再次调用了setCheckedId()方法。到这里也就明白为什么RadioGroup调用check()方法onCheckedChanged()调用多次。

##防止onCheckedChanged()调用多次方法
通过RadioButton的toggle()方法来替代RadioGroup的check()方法即可。toggle()方法也只是调用了RadioButton()的setChecked()方法而已。







