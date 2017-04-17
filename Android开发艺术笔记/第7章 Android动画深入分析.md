#第7章 Android动画深入分析

Android的动画可以分为三种：View动画、帧动画和属性动画，其实帧动画也属于View动画的一种。View动画通过对场景里的对象不断做图像变换(平移、缩放、旋转、透明度)从而产生动画效果，它是一种渐进式的动画，并且View动画支持自定义。帧动画通过顺序播放一系列图像从而产生动画效果，可以简单理解为图片切换动画，很显然，如果图片过多过大就会导致OOM。属性动画通过动态地改变对象的属性从而达到动画效果，属性动画为API11的星特性，在低版本无法直接使用属性动画，可以使用nineoldandroids库来兼容旧版本。

##7.1动画

View动画的作用对象时View,它支持4种动画效果，分别是平移动画、缩放动画、旋转动画和透明度动画。

###7.1.1View动画的种类

View动画的四种变换效果对应着Animation的四个子类：TranslateAnimation、ScaleAnimation、RotateAnimation、AlphaAnimation。这四种动画既可以通过XML来定义，也可以通过代码来动态创建，创建的文件位于res/anim/目录下。

View动画对应的XML标签`<translate>`、`<scale>`、`<rotate>`、`<alpha>`，`<set>`标签标示动画集合，对应AnimationSet类。

`<translate>`、`<scale>`的属性不仅可以用具体dp数值，还可以使用%或.标示。50%、0.5都表示控件宽度的一半例如:

```xml
<scale
    android:fromXScale="20dp"
    android:toXScale="50%"
    android:toYScale="100%"
    android:fromYScale="100%"
    android:pivotX="100%"
    android:pivotY="0.5"/>
```

在`<scale>`标签中提到了轴点的概念，举个例子，默认情况下轴点是View的中心点，这个时候在水平方向进行缩放的话会导致View向左右两个方向同时进行缩放，但是如果把轴点设为View的右边界，那么View就只会向左边进行缩放，反之则向右边进行缩放。

`<rotate>`默认情况下轴点为View的中心点。

###7.1.2自定义View动画

派生一种新动画只需要继承Animation这个抽象类，然后重写它的initialize和applayTransformation方法，在initialize方法中做一些初始化工作，在applyTransformation中进行相应的矩阵变换即可，很多时候需要采用Camera来简化矩阵变换的过程。

通常在进行3D变换时，都会用到下面两行代码:

```java
matrix.preTranslate(-centerX,-centerY);
matrix.preTranslate(centerX,centerY);
```

这两行代码的作用是将XY坐标系变换的中心点移到(centerX,centerY)，再移回到(0,0)。

关于更多内容参看文章：
https://www.github.com/wslaimin/blog/Android坐标系及矩阵变换

###7.1.3帧动画
帧动画是顺序播放一组预先定义好的图片，类似于电影播放。系统提供了另外一个类AnimationDrawable来使用帧动画，对应`<animation-list>`标签。

>注意：帧动画比较容易引起OOM,所以在使用帧动画时应尽量避免使用过多尺寸较大的图片。

##7.2LayoutAnimation
LayoutAnimation作用于ViewGroup，为ViewGroup指定一个动画，这样当它的子元素出场时都会具有这种动画效果。

（1）定义LayoutAnimation:

```xml
//  res/anim/anim_layout.xml
<layoutAnimation xmlns:android="http://schemas.android.com/apk/res/android"
    android:animationOrder="normal"
    android:animation="@anim/anim_item"
    android:delay="0.5">
</layoutAnimation>
```

android:delay:表示元素开始动画的时间延迟，比如子元素入场动画的时间周期为300ms，那么0.5表示每个子元素需要延迟150ms才能播放动画。总体来说，第一个子元素延迟150ms开始播放入场动画，第2个子元素延迟300ms开始播放入场动画，依次类推。
android:delay=0表现效果如下：
![](https://www.github.com/wslaimin/blog/raw/master/pics/nodelay.png)
android:delay=0.5表现效果如下：
![](https://www.github.com/wslaimin/blog/raw/master/pics/delay.png)

android:animationOrder:表示子元素动画的顺序，有三种normal，reverse和random。

android:animation:表示子元素的具体入场动画。

（2）为子元素指定具体的入场动画

（3）为ViewGroup指定android:layoutAnimation属性。

```xml
<ListView
        android:id="@+id/listView"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:layoutAnimation="@anim/anim_layout"
        android:background="#fff4f79f"/>
```

除了在XML中指定LayoutAnimation外，还可以通过LayoutAnimationController来实现：

```java
ListView listView=(ListView)layout.findViewById(R.id.list);
Animation animation=AnimationUtils.loadAnimation(this,R.anim.anim_item);
LayoutAnimationController controller=new LayoutAnimationController(animation);
controller.setDelay(0.5f);
controller.setOrder(LayoutAnimationController.ORDER_NORMAL);
listView.setLayoutAnimation(controller);
```

###7.2.2Activity的切换效果
Actiity有默认的切换效果，但是这个效果是可以自定义的，主要用到overridePendingTransition(int enterAnim,int exitAnim)这个方法，这个方法必须在startActivity(Intent)或者finish()之后被调用才能生效。

Fragment也可以添加切换动画，通过FragmentTransaction中的setCustomAnimations()方法来添加切换动画。

##7.3属性动画
属性动画是API11新加入的特性，属性动画可以对任何对象做动画。

###7.3.1使用属性动画
属性动画可以对任意对象的属性进行动画而不仅仅是View，动画默认时间间隔300ms，默认帧率10ms/帧。其可以达到的效果是：在一个时间间隔内完成对象从一个属性值到另一个属性值的改变。

比较常用的几个动画类是：ValueAnimator,ObjectAnimator和AnimatorSet，其中ObjectAnimator继承自ValueAnimator,AnimatorSet是动画集合。这几个类的使用：

```java
//该变一个对象的translationY属性
ObjectAnimator.ofFloat(myObject,"translationY",-myObject.getHeight()).start();

//改变一个对象的背景色属性
ValueAnimator colorAnim=ObjectAnimator.ofInt(this,"backgroundColor",oxffff8080,oxff8080ff);
colorAnim.setDuration(3000);
colorAnim.setEvaluator(new ArgbEvaluator());
colorAnim.setRepeatCount(ValueAnimator.INFINITE);
colorAnim.setRepeatMode(ValueAnimator.REVERSE);
colorAnim.start();

//动画集合
AnimatorSet set=new AnimatorSet();
set.playTogether(
    ObjectAnimator.offFloat(myView,"ratationX",0,360),
    ObjectAnimator.offFloat(myView,"ratationY",0,360),
    ObjectAnimator.offFloat(myView,"ratation",0,-90),
    ObjectAnimator.offFloat(myView,"TranslationX",0,90),
    ObjectAnimator.offFloat(myView,"TranslationY",0,90),
    ObjectAnimator.offFloat(myView,"scaleX",1,1.5f),
    ObjectAnimator.offFloat(myView,"scaleY",1,1.5f),
    ObjectAnimator.offFloat(myView,"alpha",1,0.25f,1)
);
set.setDuration(5*1000).start();
```

属性动画除了通过代码实现外，还可以通过XML来定义。属性动画需要定义在res/animator/目录下。语法如下：

```xml
<set
  android:ordering=["together" | "sequentially"]>

    <objectAnimator
        android:propertyName="string"
        android:duration="int"
        android:valueFrom="float | int | color"
        android:valueTo="float | int | color"
        android:startOffset="int"
        android:repeatCount="int"
        android:repeatMode=["repeat" | "reverse"]
        android:valueType=["intType" | "floatType"]/>

    <animator
        android:duration="int"
        android:valueFrom="float | int | color"
        android:valueTo="float | int | color"
        android:startOffset="int"
        android:repeatCount="int"
        android:repeatMode=["repeat" | "reverse"]
        android:valueType=["intType" | "floatType"]/>

    <set>
        ...
    </set>
</set>
```

在XML中可以定义ValueAnimator,ObjectAnimator以及AnimatorSet，其中<set>标签对应AnimatorSet,<animator>标签对应ValueAnimator,而<objectAnimator>则对应ObjectAnimator。下面对一些属性做说明：

 - android:ordering：有两个可选值"togerther"和"sequentially"，默认值是"together"。
 - android:startOffset:表示动画延迟时间
 - android:repeatCount:表示动画的重复次数，默认值为0，-1表示无限循环
 - android:repeatMode:表示动画的重复模式，有两个选项"restart"和"reverse"

在实际开发中建议采用代码来实现属性动画，这是因为通过代码来实现比较简单。更重要的是，很多时候一个属性的起始值是无法提前确定的。

><font color="0xff000000">注意：可以通过view的setPivotX(),setPivotY()方法改变动画的中心点坐标</font>

###7.3.2理解插值器和估值器
TimeInterpolator为时间插值器，它的作用是根据时间流逝的百分比来计算出当前属性值改变的百分比。TypeEvaluator类型估值器，它的作用是根据当前属性改变的百分比来计算改变后的属性值。

下面是插值器的工作原理：
![](https://www.github.com/wslaimin/blog/raw/master/pics/TimeInterpolator.png)

<font color="oxff000000">属性动画要求对象的该属性有set方法和get方法(可选，因为不提供属性初始值要先获取初始值)。</font>

自定义插值器要实现Interpolator或者TimeInterpolator,自定义估值器需要实现TypeEvaluator。

###7.3.3属性动画的监听器
AnimatorUpdateListener和AnimatorListener用于监听动画播放过程。AnimatorUpdateListener比较特殊，每播放一帧，onAnimationUpdate就会被调用一次。

###7.3.4对任意属性做动画
View的Scale动画效果和属性动画效果对比：
![](https://www.github.com/wslaimin/blog/raw/master/pics/scalediff.png)
可以看到View动画实际上是作用于View影像，内容会发生拉伸。

属性动画的原理：属性动画要求动画作用于的对象提供该属性的set和get方法，属性动画根据外界传递的该属性的初始值和最终值，以动画的效果多次去调用set方法，每次传递给set方法的值都不一样，确切来说是随着时间的推移，所传递的值越来越接近最终值。

想让属性动画生效，要同时满足两个条件：

 - object必须提供setAbc方法，如果动画的时候没有传递初始值，那么还要提供getAbc方法
 - object的setAbc对属性abc所做的改变必须能够通过某种方法反应出来，比如会带来UI的改变之类的

不满足上述条件的，有3中解决方法：

 - 给对象加上get和set方法，如果你有权限的话
 - 用一个类来包装原始对象，间接为其提供get和set方法
 - 采用ValueAnimator,监听动画过程，自己实现属性的改变

方法2例子：

```java
private static class ViewWrapper{
    private View mTarget;
    
    public ViewWrapper(View view){
        mTarget=view;
    }
    
    public int getWidth(){
        return mTarget.getLayoutParams().width;
    }
    
    public void setWidth(int width){
        mTarget.getLayoutParams().width=width;
        mTarget.requestLayout();
    }
}
```

方法3例子：

```java
private void performAnimate(final View target,final int start,final int end){
    ValueAnimator valueAnimator=ValueAnimator.ofInt(1,100);
    valueAnimator.addUpdateListener(new AnimatorUpdateListener(){
        private IntEvaluator mEvaluator=new IntEvaluator();
        
        @Override
        public void onAnimationUpdate(ValueAnimator animator){
            //获得当前动画的进度值，整形，1~100之间
            int currrentValue=(Integer)animator.getAnimatedValue();
            //获得当前进度占整个动画过程的比例，浮点型，0~1之间
            float fraction=animator.getAnimatedFraction();
            target.getLayoutParams().width=mEvaluator.evaluate(fraction,start,end);
            target.requestLayout();
        }
    });
    
    valueAnimator.setDuration(5000).start();
}
```

##7.4使用动画的注意事项

 - OOM问题:主要出现在帧动画中
 - 内存泄露：在属性动画中有一类无限循环的动画，这类动画需要在Activity退出时及时停止，否则将导致Activity无法释放从而造成内存泄露，通过验证后发现View动画并不存在此问题
 - 兼容性问题：动画在3.0以下的系统上有兼容性问题
 - View动画的问题：View动画是对View的影像做动画，并不是真正地改变View的状态，有时候会出现动画完成后View无法隐藏的现象，即setVisibility(View.GONE)失效了，这个时候只要调用view.clearAnimation()清除View动画即可解决此问题
 - 不要使用px
 - 动画元素的交互:将view移动后，在Android3.0以前的系统上，不管是View动画还是属性动画，新位置均无法触发单击事件，同时，老位置仍然可以触发单击事件。
 - 硬件加速：建议开启硬件加速

 
 
 
