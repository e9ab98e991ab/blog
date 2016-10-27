[TOC]
##关于App Resource
###类型
Resource有一下几种类型：
1、Animation Resources：包括属性动画、视图动画（补间动画、帧动画）。属性动画xm文件l在res/animator/目录下，如res/animator/filenam.xml；补间动画xml文件在目录res/anim/目录下，帧动画xml文件在res/drawable/目录下。
2、Color State List Resource：xml文件在res/color/目录下
3、Drawable Resources：图片或xml问价在res/drawable/目录下
4、Layout Resource：xml文件在res/layout/目录下
5、Menu Resource：xml文件在res/menu/目录下
6、String Resources：xml文件在res/values/目录下
7、Style Resource：xml文件在res/values/目录下
8、其他类型：包括Bool、Color、ID（ID在xml文件中声明，非Layout的xml文件）、Dimension等
###使用
Resource可以在xml中引用，也可以在代码中引用，使用方式有所不同。
1、在xml文件中使用：在xml文件中引用resource，语法“@type/resource_name"，如引用一个TextView“@id/textView”
2、在代码中使用：通过TextView的id（R.id.textView），获取TextView对象的引用
###原理
对于每个Resource都可以用一个整数来关联，对于Layout Resource、Drawable Resource等Resource SDK工具会在R.class自动生成一个整数对应Resource，整数的引用通常是文件名。
>android:id="@+id/resource_name"这种方式是给设置控件一个ID，同时，如果R.class文件中没有声明resource_name，则会创建resource_name的int。


