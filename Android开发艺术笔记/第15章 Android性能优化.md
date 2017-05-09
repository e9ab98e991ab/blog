# 第15章 Android性能优化

手机受内存限制会出现OOM，受CPU资源限制，会出现ANR。

##15.1 Android的性能优化方法

###15.1.1 布局优化

布局优化的思想：尽量减少布局文件的层级，层级少了，就意味着Android绘制时的工作量少了。

一般步骤：

 1. 删除布局中无用的控件和层级
 2. 有选择地使用性能较低的ViewGroup
 3. 采用<include>标签,<merge>标签和ViewStub。<merge>标签一般和<include>配合使用，ViewStub提供了按需加载的功能

<include>标签只支持以android:layout_开头的属性(android:id也支持)。如果<include>指定了id属性，同时被包含的布局文件的根元素也指定了id属性，那么以<include>指定的id属性为准。如果<include>标签指定了android:layout_*这种属性，那么要求android:layout_width和android:layout_height必须存在。

ViewStub使用：

```xml
<ViewStub
    //ViewStub的id
    android:id="@+id/stub_import"
    //layout/layotu_network_error布局根元素id
    android:inflatedId="@+id/panel_import"
    android:layout="@layout/layout_network_error"
    android:layout_width="match_parent"
    android:layout_height="match_parent"/>
```

```java
//方式一
((ViewStub)findViewById(R.id.stub_import)).setVisibility(View.VISIBLE);
//方式二
View importPanel=((ViewStub)findViewById(R.id.stub_import)).inflate();
```

###15.1.2 绘制优化

绘制优化是指View的onDraw方法要避免执行大量的操作。体现以下两方面：

 - onDraw中不要创建新的局部对象
 - onDraw方法中不要做耗时的任务。View的绘制帧率保证60fps是最佳的，这就要求每帧的绘制时间不超过16ms(1000/60)。

###15.1.3 内存优化

 - 静态变量导致的内存泄露
 - 单例模式导致的内存泄露
 - 属性动画导致的内存泄露

###15.1.4 响应速度优化和ANR日志分析

响应速度优化的核心思想是避免在主线程中做耗时的操作。

Activity如果5秒钟之内无法响应屏幕触摸事件或者键盘输入事件就会出现ANR，而BroadcastReceiver如果10秒内还未执行完操作也会出现ANR。

当进程发生ANR后，系统会在data/anr目录下创建一个文件traces.txt，通过分析这个文件就能定位ANR的原因。

###15.1.5 ListView和Bitmap优化

ListView优化：

 - 采用ViewHolder并避免在getView中执行耗时操作
 - 根据列表的滑动状态来控制任务的执行频率，比如当列表快速滑动时显然不太适合开启大量的异步任务
 - 尝试开启硬件加速

Bitmap优化：通过BitmapFactory.Options来根据需要对图片进行采样

###15.1.6 线程优化

线程优化的思想是采用线程池，避免程序中存在大量的Thread。

###15.1.7 一些性能优化建议

 - 避免创建过多对象
 - 不要过多使用枚举，枚举占用的内存空间比整型大
 - 常量使用static final来修饰
 - 使用一些Android特有的数据结构，如SpareArray和Pair等
 - 适当使用软引用和弱引用
 - 采用内存缓存和磁盘缓存
 - 尽量采用静态内部类，可以避免潜在的由于内部类而导致内存泄露

###15.2 内存泄露分析之MAT工具

转换成MAT能识别的hprof文件

```
hprof-conv inputfile outputfile
```

分析步骤

 - 打开Dominator Tree
 - 找到分析对象，右键->Path to GC Roots->exclude wake/soft reference

 
 
 
 
 
 