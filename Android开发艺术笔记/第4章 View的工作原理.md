#第4章 View的工作原理
##初始ViewRoot和DecorView
DecorView作为顶级View，一般情况下它内部会包含一个竖直方向的LinearLayout,在这个LinearLayout里面有上下两个部分(具体情况和Android版本及主题有关，上面是标题栏，下面是内容栏。)

##理解MeasureSpec
MeasureSpec在很大程度上决定了一个View的尺寸规格，之所以说是很大程度上是因为这个过程还受父容器的影响，因为父容器影响View的MeasureSpec的创建过程。

MeasureSpec代表一个32位int值，高2位代表SpecMode，低30位代表SpecSize。

SpecMode有三类：
UNSPECIFIED:父容器不对View有任何限制，要多大给多大，这种情况一般用于系统内部，表示一种测量的状态。
EXACTLY:父容器已经检测出View所需要的精确大小，这个时候View的最终大小就是SpecSize所指定的值。它对应于LayoutParams中的match_parent和具体的数值两种模式。
AT_MOST:父容器指定了一个可用大小即SpecSize，View的大小不能大于这个值，具体是什么值要看不同View的具体实现。它对应于LayoutParams中的wrap_content。
 
对于DecorView，其MeasureSpec由窗口的尺寸和其自身的LayoutParams来共同确定；对于普通View，其MeasureSpec由父容器的MeasureSpec和自身的LayoutParams来共同决定，MeasureSpec一旦确定后，onMeasure中就可以确定View的测量宽/高。

普通View的MeasureSpec的创建规则：
![](https://github.com/wslaimin/blog/raw/master/pics/MeasureSpec.jpg)

##View的工作流程
View的工作流程主要是指measure、layout、draw这三大流程，其中measure确定View的测量宽/高，layout确定View的最终宽/高和四个顶点的位置，而draw则将View绘制到屏幕上。

Drawable在什么情况下有原始宽度？ShapeDrawable无原始宽/高，而BitmapDrawable有原始宽/高。

直接继承View的自定义控件需要重写onMeasure方法并设置wrap_content时的自身大小，否则在布局中使用wrap_content就相当于使用match_parent。

为什么ViewGroup不像View一样对其onMeasure方法做统一实现呢？因为不同的ViewGroup子类有不同的布局特性，这导致它们的测量细节各不相同，比如LinearLayout和RelativeLayout这两者的布局特性显然不同，因此ViewGroup无法做统一实现。

在onMeasure方法中拿到的测量宽/高很可能是不准确的。一个比较好的习惯是在onLayout方法中去获取View的测量宽/高或者最终宽/高。

在onCreate、onStart、onResume中均无法正确得到某个View的宽/高信息，这是因为View的measure过程和Activity的生命周期方法不是同步执行的，因此无法保证Activity执行了onCreate、onStart、onResume时某个View已经测量完毕了。

四种方法获取View的宽/高：

 1. 在Activity的onWindowFocusChanged方法中获取宽/高。当Activity继续执行和暂停执行时，onWindowFocusChanged均会被调用，如果频繁地进行onResume和onPause，那么onWindowFocusChanged也会频繁被调用。
 2. view.post(runnable)。通过post可以将一个runnable投递到消息对列的尾部，然后等待Looper调用此runnable的时候，View也已经初始化好了。
 3. ViewTreeObserver。onGlobalLayoutListener接口，当View树的状态发生改变或者View树内部的View的可见性发生改变时，onGlobalLayout方法将会被回调，因此这是获取View的宽/高一个很好的时机。
 4. view.measure(int widthMeasureSpec,int heightMeasureSpec)。通过手动对View进行Measure来得到View的宽/高。这种方法要根据View的LayoutParams来分：
match_parent
直接放弃，无法measure出具体宽/高。因为根据View的measure过程，构造此种MeasureSpec需要知道parentSize，而这个时候无法知道parentSize的大小。

具体的数值(dp/px)

```java
int widthMeasureSpec=MeasureSpec.makeMeasureSpec(100,MeasureSpec.EXACTLY);
int heightMeasureSpec=MeasureSpec.makeMeasureSpec(100,MeasureSpec.EXACTLY);
view.measure(widthMeasureSpec,heightMeasureSpec);
```

wrap_content

```java
int widthMeasureSpec=MeasureSpec.makeMeasureSpec((1<<30)-1,MeasureSpec.AT_MOST);
int heightMeasureSpec=MeasureSpec.makeMeasureSpec((1<<30)-1,MeasureSpec.AT_MOST);
view.measure(widthMeasureSpec,heightMeasureSpec);
```

View的绘制过程遵循如下几步：
(1)绘制背景background.draw(canvas)。
(2)绘制自己(onDraw)。
(3)绘制children(dispatchDraw)。
(4)绘制装饰(onDrawScrollBars)。

通过setWillNotDraw可以设置WILL_NOT_DRAW标记位。 如果一个View不需要绘制任何内容，那么设置这个标记位为true后，系统会进行相应的优化

##自定义View
自定义View分类：

 1. 继承View重写onDraw方法。
 2. 继承ViewGroup派生特殊Layout。
 3. 继承特定的View(比如TextView)。
 4.继承特定的ViewGroup(比如LinearLayout)。

注意事项：

 1. 让View支持wrap_content。
 2. 如果有必要，让View支持padding。
 3. 尽量不要在View中使用Handler。post方法完全可以代替Handler的作用。
 4. View中如果有线程或者动画，需要及时停止，参考View#onDetachedFromWindow。包含此View的Activity退出或者当前View被remove时，View的onDetachedFromWindow方法会被调用。
 5. View带有滑动嵌套情形时，需要处理好滑动冲突。



