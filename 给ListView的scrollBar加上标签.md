# 给ListView的scrollBar加上标签
在使用信鸽的时候看到ListView的scorllBar加上了个标签，出于好奇，相对此一探究竟，看看是如何实现这种效果的。效果如图：

![](https://www.github.com/wslaimin/blog/raw/master/pics/scrollBar.png)

###确定算法
翻看ListView和继承父类的API，并没有发现如getScrollBarHeight()，getScrollBarOffset()之类的API，但是发现有computeVerticalScrollExtent()，computeVerticalScrollRange()，computeVerticalScrollOffset()这几个API，借助这几个API可以间接计算出scrollBar的高度和偏移量。

API介绍：
computeVerticalScrollExtent()：scrollBar滚动当前一屏内容需的偏移量
computeVerticalScrollRange()：scrollBar偏移量的范围
computeVerticalScrollOffset():scrollBar当前的偏移量

初看这几个API的注释我也感觉似懂非懂，所以，还是去源码一探究竟吧。
computeVerticalScrollRange():

```java
protected int computeVerticalScrollRange() {
    int result;
    if (mSmoothScrollbarEnabled) {
    result = Math.max(mItemCount * 100, 0);
        if (mScrollY != 0) {
            // Compensate for overscroll
            result += Math.abs((int) ((float) mScrollY / getHeight() * mItemCount * 100));
        }
    } else {
        result = mItemCount;
    }
    return result;
}
```

上面的代码计算出scrollBar的偏移量范围为itemCount*100。

computeVerticalScrollExtent()：

```java
protected int computeVerticalScrollExtent() {
    final int count = getChildCount();
    if (count > 0) {
        if (mSmoothScrollbarEnabled) {
            int extent = count * 100;
            
            View view = getChildAt(0);
            final int top = view.getTop();
            int height = view.getHeight();
            if (height > 0) {
                extent += (top * 100) / height;
            }
    
            view = getChildAt(count - 1);
            final int bottom = view.getBottom();
            height = view.getHeight();
            if (height > 0) {
                extent -= ((bottom - getHeight()) * 100) / height;
            }

            return extent;
        } else {
            return 1;
        }
    }
    return 0;
}
```

上面计算的是滚动当前一屏内容（更准确的说是当前屏幕显示的item）的偏移量。

>extent += (top * 100) / height，top是负值，要减去在屏幕顶部外面的部分。同理要减掉在屏幕底部外面的部分。

computeVerticalScrollOffset():

```java
protected int computeVerticalScrollOffset() {
    final int firstPosition = mFirstPosition;
    final int childCount = getChildCount();
    if (firstPosition >= 0 && childCount > 0) {
        if (mSmoothScrollbarEnabled) {
            final View view = getChildAt(0);
            final int top = view.getTop();
            int height = view.getHeight();
            if (height > 0) {
                return Math.max(firstPosition * 100 - (top * 100) / height +(int)((float)mScrollY / getHeight() * mItemCount * 100), 0);
            }
        } else {
            int index;
            final int count = mItemCount;
            if (firstPosition == 0) {
                index = 0;
            } else if (firstPosition + childCount == count) {
                index = count;
            } else {
                index = firstPosition + childCount / 2;
            }
            return (int) (firstPosition + childCount * (index / (float) count));
        }
    }
    return 0;
}
```

上面的代码是计算的是scrollBar的已经滚动的偏移量。

从上面三个方法的源码可以获得几个更重要的信息：

 - scrollBar的extent，range,offset均是是以item个数来计量的
 - 上面计算的并非scrollBar在屏幕上的实际表现，但是可以以此作为参考系来计算scrollBar在屏幕上的值(两者之间有相同的比例关系)如图：

![](https://www.github.com/wslaimin/blog/raw/master/pics/extent.png)
 
在屏幕上的scrollBar：
height=getMeasuredHeight()*computeVerticalScrollExtent()/computeVerticalScrollRange()

>getMeasuredHeight()为ListView测量后的高度

offset=computeVerticalScrollOffset()*(getMeasuredHeight()-heigh)/(computeVerticalScrollRange()-computeVerticalScrollExtent())

>为什么offset不等于computeVerticalScrollOffset()/computeVerticalScrollRange()?其实一般情况下，这个算法和上面计算的结果是一样的，但是系统对scrollBar的真实height做了限制，最小值为scrollBar的Drawable的宽度的2倍。(可以参考ScrollBarUtils和ScrollBarDrawable)

上面是分析算法，项目中的话可以用ScrollBarUtils更方便，也更准确。

附上例子：




















