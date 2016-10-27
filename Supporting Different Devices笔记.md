[TOC]
##支持不同语言
Android平台能够在运行时根据本地区域设置来选择不同语言。如果所有string都来源strings.xml，那么定义可选的string.xml文件，android系统在运行时会进行正确选择。

实现方法：
默认创建的string.xml在res/values/目录下。为支持不同语言，在res/目录下创建包含"-"和ISO语言码的values目录，形如"values-es"(es为ISO语言码)。完整目录结构如下：
MyProject/
　res/
　　values/
　　　strings.xml
　　values-es/
　　　strings.xml
　　values-fr/
　　　strings.xml
##支持不同屏幕
Android设备以两个属性进行分类：size(尺寸)、density(像素密度)。

size分为：small、normal、large、xlarge、landscape、portrait
density分为：low(ldpi)、medium(mdpi)、hight(hdpi)、extra hight(xhdpi)
随着手机越来越大，分辨率越来越高，size和density可能不止上面几种。

适配不同size和density屏幕的机制和多语言支持是一样的，主要就是为不同的size提供可选layout，为不同density提供可选drawable。

支持不同size方法：
在res/目录下创建以"-<screen_size>"为后缀的layout目录，形如"res/layout-large"。布局文件名必须是一致的。完整目录结构如下：
MyProject/
　res/
　　layout/
　　　main.xml
　　layout-large/
　　　main.xml
>注意：android会自动缩放布局来正确适应屏幕（dp为单位的尺寸与像素密度无关）。因此，重心应该放在布局结构而不是UI元素的绝对尺寸上。

默认layout/main.xml用于竖屏。
如果想为large屏幕和landscape提供一个layout，可以组合使用：
res/layout-large-land/main.xml
>注意：Android 3.2和以上版本提供了一种高级方法来支持不同屏幕大小。https://developer.android.com/training/multiscreen/index.html

density分类及相关比例：
MDPI : 160 DPI
HDPI = 1.5 x MDPI = 240 DPI
XHDPI = 2 x MDPI = 320 DPI
XXHDPI = 3 X MDPI = 480 DPI
XXXHDPI = 4 X MDPI = 640 DPI
在MDPI下1dp=1px

支持不同density方法：在相应drawable文件夹下放不同分辨率图片。目录结构如下：
MyProject/
　res/
　　drawable-xhdpi/
　　　awesomeimage.png
　　drawable-hdpi/
　　　awesomeimage.png
　　drawable-mdpi/
　　　awesomeimage.png
　　drawable-ldpi/
　　　awesomeimage.png
>注意：可以不用在ldpi资源不是必须的。当有更高分辨率资源时，系统会自动缩小图片来适应ldpi屏幕。因此，只需准备需要支持的最大分辨率屏幕的图片即可。
##支持不同平台版本
由于Android是开源的，导致版本碎片化严重，开发过程要尽可能兼容市面上大多数版本。
>注意：为了旧版本系统也能够使用最新发布版本的特性，Google会同时推出向下兼容的Android Support Library。旧版本使用这个包即可
使用新版本的Api。

在AdroidManifest.xml文件会指定两个属性：minSdkVersion,targetSdkVersion。minSdkVersion表示app兼容的最低api级别，targetSdkVersion表示app设计和测试使用的最高api级别。为了使app能够使用新版本特性以及适应大多数手机设备，targetSdkVersion应该紧跟最新Android版本。

有些代码依赖高版本api，可以在运行进行系统版本检查：
```java
private void setUpActionBar() {
    // Make sure we're running on Honeycomb or higher to use ActionBar APIs
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.HONEYCOMB) {
        ActionBar actionBar = getActionBar();
        actionBar.setDisplayHomeAsUpEnabled(true);
    }
}
```
>注意：在XML文件里可以放心使用新属性，低版本会直接忽略新属性。

给Activity指定主题：
```xml
<activity android:theme="@android:style/Theme.Dialog">
```
给所有Activity指定主题：
```xml
<application android:theme="@style/CustomTheme">
```