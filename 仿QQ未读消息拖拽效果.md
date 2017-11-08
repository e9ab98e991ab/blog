# 仿QQ未读消息拖拽效果

## 技术参考文章

<a href="http://www.html-js.com/article/1628">贝塞尔曲线介绍</a>

<a href="http://blog.csdn.net/oqinyou/article/details/65444808?1490316718116">Android自定义控件：类QQ未读消息拖拽效果</a>

### 补充说明

<a href="http://blog.csdn.net/oqinyou/article/details/65444808?1490316718116">Android自定义控件：类QQ未读消息拖拽效果</a>，这篇文章在求解A,B,C,D四个点上不尽详述，所以下面进行补充说明。

图示：

![](https://www.github.com/wslaimin/blog/raw/master/pics/qqdot_caculate.png)

计算过程：

![](https://www.github.com/wslaimin/blog/raw/master/pics/qqdot_abcd.png)


## 技术总结

 1. 计算A,B,C,D四点坐标
 2. 了解拖拽过程连接固定圆和拖拽圆之间的曲线实现，即贝塞尔曲线
 3. 如何让QQDotView拖拽区域为整个屏幕？使用WindowManager的addView方法，把QQDotView添加到Window上。
 4. WindowManager.LayoutParams设置format为PixelFormat.TRANSLUCENT，背景为半透明。View添加到特定坐标位置时，需要设置gravity=Gravity.LEFT | Gravity.TOP,否则View添加的位置会不正确。
 4. 使用事件代理，将固定点的触摸事件代理给QQDotView。
 5. 在ListView中拖动圆点ListView可能会发生滚动，在QQDot按下时调用requestDisallowInterceptTouchEvent(true)，不允许父容器拦截事件。
 6. 在ListView中由于View复用，在滚动后固定点坐标没有更新。重写QQDot的dispatchTouchEvent(MotionEvent ev)方法，更新固定点坐标。
 7. 回弹动画，使用translationX,translationY属性动画。
 8. 消失动画使用帧动画。
  
 
