#ListView设置选中状态
##使用方法
通常在ListView的子View被选中时，希望给顶一个被选中的状态，比如，更改背景色。

为了使子View在选中时改变背景，可以用`<selector/>`标签实现。
activated.xml

```xml
<selector xmlns:android="http://schemas.android.com/apk/res/android">
    <item android:state_activated="true" android:drawable="@color/yellow"/>
    <item android:drawable="@color/black"/>
</selector>
```

>注意：使用的是android:state_activated属性，非android:checked、android:selected。为什么使用这个属性，后面源码分析会介绍。

item.xml

```xml
<TextView xmlns:android="http://schemas.android.com/apk/res/android"
    android:id="@android:id/text1"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:gravity="center_vertical"
    android:background="@drawable/activated"/>
```

要让ListView能够显示选中状态，只需给ListView设置选中模式即可。

```java
getListView().setChoiceMode(ListView.ListView.CHOICE_MODE_SINGLE);
```

到这里，ListView设置选中状态就完成了。点击ListView的子View会发现选中的View背景颜色发生了改变。

##源码分析
以下源码基于API 21。只针对单选模式，多选模式的实现也相类似。

首先，从ListView的performItemClick方法开始分析。

```java
......
else if (mChoiceMode == CHOICE_MODE_SINGLE) {
    boolean checked = !mCheckStates.get(position, false);
    if (checked) {
        mCheckStates.clear();
        mCheckStates.put(position, true);
        if (mCheckedIdStates != null && mAdapter.hasStableIds()) {
            mCheckedIdStates.clear();
            mCheckedIdStates.put(mAdapter.getItemId(position), position);
        }
    mCheckedItemCount = 1;
    } else if (mCheckStates.size() == 0 || !mCheckStates.valueAt(0)) {
        mCheckedItemCount = 0;
    }
    checkedStateChanged = true;
}

if (checkedStateChanged) {
    updateOnScreenCheckedViews();
}
```

选择模式是单选模式下，点击位置由未选中状态变为选中状态，mCheckStates会清空之前选中position的状态，以当前position为key，true为value存入mCheckStates，并且mCheckedItemCount设置为1；当选中的是同一position，mCheckStates不改变，mCheckedItemCount不改变。之后会调用updateOnScreenCheckedViews方法。updateOnScreenCheckedViews方法的实现。

```java
private void updateOnScreenCheckedViews() {
    final int firstPos = mFirstPosition;
    final int count = getChildCount();
    final boolean useActivated = getContext().getApplicationInfo().targetSdkVersion
>= android.os.Build.VERSION_CODES.HONEYCOMB;
    for (int i = 0; i < count; i++) {
        final View child = getChildAt(i);
        final int position = firstPos + i;
        if (child instanceof Checkable) {
            ((Checkable) child).setChecked(mCheckStates.get(position));
            } else if (useActivated) {
                child.setActivated(mCheckStates.get(position));
            }
        }
}
```

targetSdkVersion>=11(android3.0)使用activated作为选中状态，因此，targetSdkVersion>11时，子View使用android:activated属性的原因。getChildCount返回当前可见View数量，遍历当前可见children，根据mCheckState存储位置的状态来设置child的状态。如果child实现Checkable接口，child调用setChecked方法，否则，如果targetSdkVersion>=11，child调用setActivated方法。child背景就会根据设置的activated状态进行改变了。

>注意：如果child是ViewGroup实例，那么activated状态会向children传递。selected状态也会向下传递。

实现过程十分简单，Demo就不贴了。


