# 从Android角度看MVC、MVP、MVVM

##名词解释
业务逻辑：对数据访问操作的简单的封装(https://blog.csdn.net/horkychen/article/details/45050969)

>我的通俗理解是对数据访问接口的访问过程。有连个步骤，第一个是是接口参数准备，第二个是调用接口访问

Model:是应用程序中用于处理应用程序数据逻辑的部分
View:是应用程序中处理数据显示的部分
Cotroller:是应用程序中处理用户交互的部分

>https://baike.baidu.com/item/MVC%E6%A1%86%E6%9E%B6/9241230?fr=aladdin&fromid=713147&fromtitle=MVC%E6%A8%A1%E5%BC%8F

##对比

1、MVC

![](https://github.com/wslaimin/blog/raw/master/pics/mvc.png)

>各层之间的调用关系比如View可以调用Model的方法，Model也可以调用View的方法

对应角色：

 - M:Model
 - V:xml文件
 - C:Activity或Fragment

缺陷：View和Controller耦合(View修改Controller也需要修改)

2、MVP

![](https://github.com/wslaimin/blog/raw/master/pics/mvp.png)

>>各层之间的调用关系

对应角色：

 - M:Model
 - V:Activity或Fragment
 - P:presenter

优点：
1、相较MVC，Presenter作为View和Modle之间的桥梁，使业务逻辑和View隔离
2、View和Presenter之间通过接口引用，Presenter不依赖具体View，可达到重用的目的

缺点：View的接口粒度不好把握。设计的太细，容易使Presenter和View形成一对一关系，导致Presenter重用性降低；太粗，Presenter对View的渲染能力又降低。

3、MVVM

![](https://github.com/wslaimin/blog/raw/master/pics/mvvm.png)

>View和ViewModel之间的箭头表示双向数据绑定，自动同步

对应角色：

M:Model
V:View
VM:ViewModel

优点：
1、View能够感知数据变化，并作出变化
2、ViewModel不依赖View，彻底与View解耦

>在我看来，ViewModel和MVP中的Presenter的作用是一样的，区别在于View和ViewModel之间采用数据绑定的交互方式。
