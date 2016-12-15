#第3章 View的事件体系
##View基础知识
View是一种界面层的控件的一种抽象，它代表了一个控件。

View的位置主要由它的四个顶点来决定，分别对应View的四个属性：top、left、right、bottom。在Android中，x轴和y轴的正方向分别为右和下。

从Android3.0开始，View增加了额外几个参数：x、y、translationX和translationY，x和y是View左上角坐标，而translationX和translationY是View左上角相对于父容器的偏移量。

View在平移过程中，top和left表示的是原始左上角的位置信息，其值并不会发生改变，此时发生改变的是x、y、translationX和translationY这四个参数。

getX/getY返回的是相对于当前View左上角的x和y坐标，而getRawX/getRawY返回的是相对于手机屏幕左上角的x和y坐标。

TouchSlop是系统所能识别出的被认为是滑动的最小距离。可以通过ViewConfiguration.get(getContext()).getScaleTouchSlop()获取。

速度的计算公式：速度=(终点位置-地点位置)/时间段
在View的onTouchEvent方法中追踪当前单击事件的速度：

```java
VelcocityTracker velcocityTracker=VelcocityTracker.obtain();
velcocityTracker.addMovement(event);
velcocityTracker.computeCurrentVelcocity(1000);
int xVelcocity=(int)velcocityTracker.getXVelocity();
int yVelcocity=(int)velcocityTracker.getYVelocity();
```

GestureDetector的用法：

```java
GestureDetector mDetector=new GestureDetector(this);
//解决长按屏幕后无法拖动的现象
mDetector.setIsLongpressEnabled(false);
```

在View的onTouchEvent方法中添加：

```java
return mDetector.onTouchEvent(event);
```

>注意：onDoubleTap方法和onSingleTapConfirmed不能共存。如果只监听滑动相关，建议自己在onTouchEvent中实现，如果要监听双击这种行为的话，那么就使用GestureDetector。

当使用View的scrollTo/scrollBy方法来进行滑动时，其过程是瞬间完成的，这个没有过渡效果的滑动用户体验不好，可以使用Scroller来实现有过渡效果的滑动。Scroller的用法：

```java
Scroller scroller=new Scroler(mContext);

private void smoothScrollTo(int destX,int destY){
    int scrollX=getScrollX();
    int delta=destX-scrollX;
    //1000ms内滑向destX，效果就是慢慢滑动
    mScroller.startScroll(scrollX,0,delta,0,1000);
    invalidate();
}

@Override
public void computeScroll(){
    if(mScroller.computeScrollOffset()){
        scrollTo(mScroller.getCurrX(),mScroller.getCurrY());
        postInvalidate();
    }
}
```

##View的滑动
通过三种方式可以实现View的滑动：第一种是通过View本身提供的scrollTo/scrollBy方法来实现滑动；第二种是通过动画给View施加平移效果来实现滑动；第三种是通过改变View的LayoutParams使得View重新布局从而实现滑动。

mScrollX的值总是等于View左边缘和View内容左边缘在水平方向的距离，而scrollY的值总是等于View上边缘和View内容上边缘在竖直方向的距离。scrollTo和scrollBy只能改变View内容的位置而不能改变View在布局中的位置。

移动可以理解为以内容为准的坐标系发生了改变。mScrollX和mScrollY的变换规律：
![](https://github.com/wslaimin/blog/raw/master/pics/scroll.jpg)

View动画是对View的影像做操作，它并不能真正改变View的位置参数，包括宽/高，并且如果希望动画后的状态得以保留还必须将fillAfter属性设置为true。补间动画还存在个问题，移动后的影像不能响应事件，原位置仍可以响应事件，属性动画可以解决这个问题。

三种方式对比：

 - scrollTo/scrollBy：操作简单，适合对View内容滑动
 - 动画：操作简单，主要使用于没有交互的View和实现复杂的动画效果
 - 改变布局参数：操作稍微复杂，适用于有交互的View
 
##弹性滑动
弹性滑动的思想：将一次大的滑动分成若干次小的滑动并在一个时间段内完成。

使用延时策略的核心思想：通过发送一系列延时消息从而达到一种渐进式的效果，具体来说可以使用Handler或View的postDelayed方法，也可以使用线程的sleep方法。
 
##View的事件分发机制
点击事件的分发过程由三个很重要的方法来共同完成：dispatchTouchEvent、onInterceptTouchEvent、onTouchEvent。

给View设置OnTouchListener，其优先级比onTouchEvent要高。OnClickListener优先级最低。

当一个点击事件产生后，它的传递过程遵循如下顺序：Activity->Window->View。

一些结论：

 1. 同一个事件序列是指从手指接触屏幕的那一刻起，到手指离开屏幕的那一刻结束。这个事件序列以down事件开始，中间含有数量不定的move事件，最终以up事件结束。
 2. 一个事件序列只能被一个View拦截且消耗。
 3. 某个ViewGroup一旦决定拦截，那么这个事件序列都只能由它来处理，并且它的onInterceptTouchEvent不会再被调用。
 4. 某个View一旦开始处理事件，如果它不消耗ACTION_DOWN事件(onTouchEvent返回了false)，那么同一事件序列中的其他事件都不会再交给它来处理，并且事件将重新交由它的父元素去处理，即父元素的onTouchEvent会被调用。
 5. 如果View不能消除ACTION_DOWN以外的其他事件，那么这个点击事件会消失，此时父元素的onTouchEvent并不会被调用，并且当前View可以持续受到后续的事件，最终这些消失的点击事件会传递给Activity处理。
 6. ViewGroup默认不拦截任何事件。
 7. View没有onInterceptTouchEvent方法，一旦有点击事件传递给它，那么它的onTouchEvent方法就会被调用。
 8. View的onTouchEvent默认都会消耗事件，除非它是不可点击的。
 9. View的enable属性不影响onTouchEvent的返回值。
 10. onClick会发生的前提是当前View是可点击的，并且它收到了down和up的事件。
 11. 事件传递过程是由外向内的，即事件总是先传递给父元素，然后再由父元素分发给子View，通过requestDisallowInterceptTouchEvent方法可以在子元素中干预父元素的事件分发过程，但是ACTION_DOWN事件除外。

事件分发源码分析：

