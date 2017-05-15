# 第8章 理解Window和WindowManager

Window表示一个窗口的概念。Window是一个抽象类，它的具体实现是PhoneWindow。WindowManager是外界访问Window的入口，Window的具体实现位于WindowManagerService中，WindowManager和WindowManagerService的交互是一个IPC过程。Android中所有的视图都是通过Window来呈现的，他们的视图实际上都是附加在Window上的，因此，Window实际是View的直接管理者。

##8.1 Window和WindowManager
通过WindowManager添加Window的过程:

```java
WindowManager windowManager=(WindowManager) getSystemService(Context.WINDOW_SERVICE);
Button button=new Button(this);
button.setText("this is a button");
button.setOnClickListener(new View.OnClickListener() {
    @Override
    public void onClick(View v) {
        System.out.println("click");
        Intent intent=new Intent(v.getContext(), RemoteActivity.class);
        startActivity(intent);
    }
});
WindowManager.LayoutParams layoutParams=new WindowManager.LayoutParams(WindowManager.LayoutParams.WRAP_CONTENT,
                WindowManager.LayoutParams.WRAP_CONTENT,0,0, PixelFormat.TRANSPARENT);
layoutParams.flags= WindowManager.LayoutParams.FLAG_NOT_TOUCH_MODAL
                | WindowManager.LayoutParams.FLAG_NOT_FOCUSABLE
                | WindowManager.LayoutParams.FLAG_SHOW_WHEN_LOCKED;
layoutParams.gravity= Gravity.LEFT|Gravity.TOP;
layoutParams.x=100;
layoutParams.y=300;
windowManager.addView(button,layoutParams);
```

Flags参数表示Window的属性有几个常用的选项：

 - FLAG_NOT_FOCUSABLE:表示Window不需要获取焦点，也不需要接收各种输入事件，此标记会同时启用FLAG_NOT_TOUCH_MODAL,最终事件会直接传递个下层的具有焦点的Window。
 - FLAG_NOT_TOUCH_MODAL:在此模式下，系统会将当前Window区域以外的单击事件传递给底层的Window，当前Window区域以内的单击事件则自己处理。
 - FLAG_SHOW_WHEN_LOCKED:开启此模式可以让Window显示在锁屏的界面上。

Type参数表示Window的类型，Window有三种类型，分别是应用Window,子Window和系统Window。应用类Window对应着一个Activity。子Window不能单独存在，它需要附属在特定的父Window之中，比如Dialog。系统Window是需要声明权限才能创建的Window，比如Toast和系统状态栏。

Window是分层的，每个Window都有对应的z-ordered，层级大的会覆盖在层级小的Window上面。应用Window的层级范围是1~99，子Window的层级范围是1000~1999，系统Window的层级范围是2000~2999。这些层级范围对应着WindowManager.LayoutParams的type参数。

##8.2 Window的内部机制
###8.2.1 Window的添加过程
Window的添加过程需要通过WindowManager的addView来实现。

 - 检查参数是否合法，如果是子Window那么还需要调整一些布局参数
 - 创建ViewRootImp并将View添加到列表中
 - 通过ViewRootImpl来更新界面并完成Window的添加过程：这个过程由ViewRootImp的setView方法来完成，在setView内部会通过requestLayout来完成异步刷新请求，接着通过WindowSession最终来完成Window的添加过程。

###8.2.2 Window的删除过程
 WindowManager中提供了两种删除接口removeView和removeViewImmediate,他们分别表示异步删除和同步删除，其中remvoeViewImmediate使用起来需要特别注意，一般来说不需要使用此方法来删除Window以免发生意外错误。
 
 - removeView()
 - removeViewLocked()
 - die()
 - 同步删除doDie()，异步删除发送MSG_DIE消息
 - doDie()内部调用dispatchDetachedFromWindow方法
 - dispatchDetachedFromWindow方法内部调用WindowSession.remove(mWindow);

###8.2.3 Window的更新过程

 - 调用WindowManager的updateViewLayout方法
 - 调用WindowManagerGlobal的updateViewLayout方法
 - 调用root.setLayoutParams()方法
 - 调用scheduleTraversals()方法，开始重新测量，布局，绘制
 
##8.3 Window的创建过程

###8.3.1 Activity的Window创建过程

Window创建流程：
 ![](https://www.github.com/wslaimin/blog/raw/master/pics/addWindow.png)
 
PhoneWindow的setContentView方法步骤：

 1. 如果没有DecorView,那么就创建它

DercorView是一个FrameLayout，DecorView是Activity中的顶级View，一般来说它的内部包含标题栏和内部兰，但是这个会随着主题的变换而发生改变。内容栏是一定要存在的，并且内容栏具有固定的id，就是“content”,它的完整id是android.R.id.content。

 2. 将View添加到DecorView的mContentParent中

 3. 回调Activity的onContentChanged方法通知Activity视图已经发生改变

Activity的布局文件已经成功添加熬了DercorView的mContentParent中，但是这个时候DecorView还没有被WindowManager正式添加到Window中。在ActivityThread的handleResumeActivity方法中，首先会调用Activity的onResume方法，接着会调用Activity的makeVisible()。

大致流程：

![](https://www.github.com/wslaimin/blog/raw/master/pics/createWindow.png)
 
 ###8.3.2 Dialog的Window创建过程
 
 Dialog的Window的创建过程和Activity类似。
 
 - 创建Window
 - 初始化DercorView并将Dialog的视图添加到DercorView中
 - 将DecorView添加到Window中并显示

普通的Dialog有一个特殊之处，那就是必须采用Activity的Context，如果采用Application的Context,那么就会报错。

系统Window比较特殊，它可以不需要token，但是使用时需要在AndroidManifest文件中声明权限。

###8.3.3 Toast的Window创建过程

在Tost的内部有两类IPC过程，第一类是Toast访问NotificationManagerService，第二类是NotificationManagerService回调Toast里的TN接口。

当NotificationManagerService处理Toast的显示或隐藏请求时会跨进程回调TN中的方法，这个时候由于TN运行在Binder线程池中，所以需要通过Handler将其切换到当前线程中。这里的线程是指发送Toast请求所在的线程。

流程：
![](https://www.github.com/wslaimin/blog/raw/master/pics/toast.png)
 
 
