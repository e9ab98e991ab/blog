#第6章 Android的Drawable笔记

Drawable表示的是一种可以在Canvas上进行绘制的抽象的概念。最常见的颜色和图片都可以是一个Drawable。

Drawable在开发中的有点：首先，它使用简单，比自定义View的成本要低；其次，非图片类型的Drawable占用空间较小，这对减小apk的大小也很有帮助。

##6.1 Drawable简介
通过getIntrinsicWidth和getIntrinsicHeight这两个方法获取的是固有宽/高。并不是所有的Drawable都有固有宽/高。比如一张图片所形成的Drawable，它的固有宽/高就是图片宽/高，但是一个颜色所形成的Drawable，就没有固有宽/高。Drawable的固有宽/高不等同于它的大小，当用作View的背景时，Drawable会被拉伸至View的同等大小。

##6.2 Drawable的分类
###6.2.1 BitmapDrawable
android:antialias:是否开启抗锯齿，开启后会让图片变得平滑，同时会在一定程度上降低图片的清晰度。

android:dither:是否开启抖动效果。当图片的像素配置和手机屏幕的像素配置不一致时，开启这个选项可以让高质量的图片在低质量的屏幕上还能保持较好的显示效果，比如图片的色彩模式为ARGB8888，但设备屏幕所支持的色彩模式为RGB555，这个时候开启抖动选项可以让图片显示不会过于失真。

android:filter:是否开启过滤效果。当图片尺寸被拉伸或者压缩时，开启过滤效果可以保持较好的显示效果。
android:gravity:当图片小于容器尺寸时，设置此选项可以对图片进行定位。不同选项可以通过"|"来组合使用。

android:mipMap:纹理映射，默认为false，不常用。
android:titleMode:平铺模式，有如下几个值：["disabled"|"clamp"|"repeat"|"mirror"]。
###6.2.2 ShapeDrawable
android:shape:有以下四个选项：rectangle、oval、line、ring。默认是rectangle，line和ring这两个选项必须要通过<stroke>标签来指定线的宽度和颜色等信息，android:useLevel必须设为false，否则无法达到预期显示效果。

针对ring这个形状，有5个特殊的属性如下：
![](https://www.github.com/wslaimin/blog/raw/master/pics/ring.jpg)

<corners>只适用于矩形shape。它的android:radius表示弧度。

<gradient>与<solid>标签是相互排斥的，gradient有如下属性：

 - android:angel：渐变角度，默认为0，其值必须为45的倍数，0表示从左到右，90表示从下到上。
 - android:centerX:渐变的中心点的横坐标。相对X的渐变位置，一般取值(0,1)。
 - android:centerY:渐变的中心点纵坐标。
 - android:startColor:渐变的起始色。
 - android:centerColor:渐变的中间色。
 - android:endColor:渐变的结束色。
 - android:gradientRadius:渐变半径，仅当android:type="radial"有效。从startColor渐变到endColor的半径。
 - android:useLevel:一般为false，当Drawable作为StateListDrawable使用时为true。
 - android:type:类别，有linear(线性渐变)、radial(径向渐变)、sweep(扫描线渐变)。

<stroke>描边。当android:dashWidth和android:dashGap有任何一个为0，那么虚线效果将不能生效。

<padding>表示空白，表示View内容离边界的距离。

<size>使用size时，shape有固有大小。

###6.2.3 LayerDrawable

LayerDrawable对应的XML标签是<layer-list>，它表示一种层次化的Drawable集合，通过将不同的Drawable放置在不同的层上面从而达到一种叠加后的效果。

android:top、android:bottom、android:left和android:right，它们分别表示Drawable相对于View的上下左右的偏移量，单位为像素。

###6.2.4 SateListDrawable

StateListDrawable对应于<selector>标签，它也是表示Drawable集合，每个Drawable都对应着View的一种状态，这样系统就会根据View的状态来选择合适的Drawable。

android:constantSize表示StateListDrawable的固有大小是否不随着其状态的改变而改变，因为状态的改变会导致StateListDrawable切换到具体的Drawable，而不同的Drawable具有不同的固有大小。

android:dither表示是否开启抖动效果，开启此选项可以让图片在低质量的屏幕上仍然获得较好的显示效果。默认值为true。

android:variablePadding表示StateListDrawable的padding是否随着其状态的改变而改变。默认值为false，并且不建议开启此选项。经测试没发现开启与不开启有何不同。

###6.2.5 LevelListDrawable

LevelListDrawable对应于<level-list>标签，它同样表示一个Drawable集合，集合中的每个Drawable都有一个等级(level)的概念。根据不同的等级，LevelListDrawable会切换为对应的Drawable。

android:minLevel和android:maxLevel指定等级范围，在最小值和最大值之间的等级会对应此item中的Drawable。Drawable的等级是有范围的，即0~10000，最小等级是0，这也是默认值，最大等级是10000。

###6.2.6 TransitionDrawable

TransitionDrawable对应于<transition>标签，它用于实现两个Drawable之间的淡入淡出效果。

开启淡入淡出效果：
```java
TransitionDrawable drawable = (TransitionDrawable)textView.getBackground();
drawable.startTransition(1000);
```

###6.2.7 InsetDrawable

InsetDrawable对应于<inset>标签，它可以将其他Drawable内嵌到自己当中，并可以在四周留出一定的间距。当一个View希望自己的背景比自己的实际区域小的时候，可以采用InsetDrawable来实现。

###6.2.8 ScaleDrawable

ScaleDrawable对应于<scale>标签，它可以根据自己的等级(level)将指定的Drawable缩放到一定比例。

android:scaleGravity的含义等同于shape中的android:gravity，而android:scaleWidth和android:scaleHeight分别表示对指定Drawable宽和高的缩放比例，以百分比的形式表示，比如25%。

等级0表示ScaleDrawable不可见，这是默认值，想要ScaleDrawable可见，需要等级不能为0，这一点从源码中可以得出。

```java
public void draw(Canvas canvas) {
    final Drawable d = getDrawable();
    if (d != null && d.getLevel() != 0) {
        d.draw(canvas);
    }
}
```

在onBoundsChange方法中，可以看出mDrawable的大小和等级以及缩放比例的关系，这里拿宽度来说。

```java
int w = bounds.width();
if (mState.mScaleWidth > 0) {
    final int iw = min ? d.getIntrinsicWidth() : 0;
    w -= (int) ((w - iw) * (MAX_LEVEL - level) * mState.mScaleWidth / MAX_LEVEL);
}
```

iw是mDrawable的固有大小，MAX_LEVEL为10000，由于一般都为0，所以上面代码可以简化为：w -= (int) (w * (MAX_LEVEL - level) * mState.mScaleWidth / MAX_LEVEL)。由此可见，如果ScaleDrawable的级别为最大值10000，那么就没有缩放效果；如果ScaleDrawable的级别(level)越大，那么内部的Drawable看起来越大；如果ScaleDrawable的XML中所定义的缩放比例越大，那么内部的Drawable看起来越小。

android:scaleWidth设为70%，android:scaleHeight设为70%,level设为1，那么可以近似地将一张图片缩小为原大小的30%。

###6.2.9 ClipDrawable

ClipDrawable对应于<clip>标签，它可以根据自己当前的等级(level)来裁剪另一个Drawable，裁剪方向可以通过android:clipOrientation和android:gravity这两个属性来共同控制。

gravity的各种选项是可以通过"|"来组合使用的。

![](https://github.com/wslaimin/blog/raw/master/pics/clipdrawable.jpg)

对于ClipDrawable来说，等级0表示完全裁剪，即整个Drawable都不可见了，而等级10000表示不裁剪。

##6.3 自定义Drawable

Drawable的工作原理很简单，其核心就是draw方法。

通常我们没有必要去自定义Drawable，因为自定义的Drawable无法在XML中使用，这就降低了自定义Drawable的使用范围。自定义Drawable如下：

```java
public class CustomDrawable extends Drawable{
    private Paint mPaint;

    public CustomDrawable(int color){
        mPaint=new Paint(Paint.ANTI_ALIAS_FLAG);
        mPaint.setColor(color);
    }

    @Override
    public void draw(Canvas canvas) {
        Rect r=getBounds();
        float cx=r.exactCenterX();
        float cy=r.exactCenterY();
        canvas.drawCircle(cx,cy,Math.min(cx,cy),mPaint);
    }

    @Override
    public void setAlpha(int alpha) {
        mPaint.setAlpha(alpha);
        invalidateSelf();
    }

    @Override
    public void setColorFilter(ColorFilter colorFilter) {
        mPaint.setColorFilter(colorFilter);
    }

    @Override
    public int getOpacity() {
        return PixelFormat.TRANSLUCENT;
    }
}
```

另外getIntrinsicWidth和getIntrinsicHeight这两个方法需要注意下，当定义的Drawable有固有大小时最好重写这两个方法，因为它会影响到View的wrap_content布局。需要注意的是，内部大小不等于Drawable的实际区域大小，Drawable的实际区域大小可以通过它的getBounds方法来得到，一般来说它和View的尺寸相同。
