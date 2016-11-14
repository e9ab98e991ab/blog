#Activiyt的生命周期
##Activity生命周期方法
onRestart:当当前Activity从不可见重新变为可见状态时，onRestart就会被调用。
onStart:Activity已经可见了，但是还没有出现在前台，还无法和用户交互。
onResume:出现在前台并开始活动。
onPause:正常情况下，紧接着onStop就会被调用。在特殊情况下，如果这个时候快速地再回到当前Activity，那么onResume会被调用。onPause必须执行完，新Activity的onResume才会执行。当用户打开新的Activity或者切换到桌面的时候，回调如下：onPause->onStop。有一种特殊情况，如果新Activity采用了透明主题，那么当前Activity不会回调onStop。

onStart和onResume，onPause和onStop的区别：onStart和onStop是从Activity是否可见这个角度来回调的，而onResume和onPause是从Activity是否位于前台这个角度来回调的。

我们知道onPause和onStop都不能执行耗时的操作，尤其是onPause，这也意味着，我们应当尽量在onStop中做操作，从而使得新Activity尽快显示出来并切换到前台。
##Activity状态保存
当系统配置发生改变后，Activity会被销毁，其onPause、onStop、onDestroy均会被调用，并在onStop之前调用onSaveInstanceState来保存当前Activity的状态，它既可能在onPause之前调用，也可能在onPause之后调用。onReStoreInstanceState在onStart之后调用。

当Activity在异常情况下需要重新创建时，系统默认为我们保存当前Activity的视图结构，并且在Activity重启后为我们恢复这些数据，比如文本框中用户输入的数据、ListView滚动的位置等，前提是View有指定id，没有指定id，系统不会做保存。

关于保存和恢复View层次结构，系统的工作流程是这样的：首先Activity被意外终止时，Activity会调用onSaveInstanceState去保存数据，然后Activity会委托Window去保存数据，接着Window再委托它上面的顶级容器去保存数据，顶层容器是一个ViewGroup，一般来说它很可能是DecorView。最后顶层容器再去一一通知它的子元素来保存数据。
由此，可以得出Activity的层级关系：
![](https://github.com/wslaimin/blog/raw/develop/pics/pic.jpg)

按Home键或者启动新Activity也会调用onSaveInstanceState，因为如果在新Activity屏幕配置发生改变，再回到之前Activity会调用onRestoreInstanceState。

Activity按照优先级从高到低，可以分为如下三种：
 (1)前台Activity——正在和用户交互的Activity，优先级最高。
 (2)可见但非前台Activity——比如Activity中弹出了一个对话框，导致Activity可见但是位于后台无法和用户直接交互。
 (3)后台Activity——已经被暂停的Activity，比如执行了onStop，优先级最低。
 当系统内存不足时，系统就会按照上述优先级先去杀死目标Activity所在的进程，并在后续通过onSaveInstanceState和OnRestoreInstanceState来存储和恢复数据。如果一个进程中没有四大组件在执行，那么这个进程将会很快被系统杀死。
 
 当系统配置发生改变后，不想系统重新创建Activity，可以给Activity指定configChanges属性。如下所示：
 ```xml
 android:configChanges="orientation"
 ```
 书中只指定"orientation"在Android5.1中测试Activity依旧会重新创建，改为```android:configChanges="orientation|screenSize"```则不会。screenSize属性是API13新添加，当minSdkVersion和targetSdkVersion均低于13时，此项不会导致Activity重启，否则会导致Activity重启。
 configChanges的项目表：
 ![](https://github.com/wslaimin/blog/raw/develop/pics/configChanges.jpg)
##Activity的启动模式
任务栈是一种“后进先出”的栈结构，当栈中无任何Activity的时候，系统就会回收这个任务栈。

有四种启动模式：standard,singleTop,singleTask,singleInstance。
(1)standard:标准模式，这也是系统的默认模式。每次启动一个Activity都会创建一个新的实例，不管这个实例是否已经存在。在这种模式下，谁启动了这个Activity，那么这个Activity久运行在启动它的那个Activity所在的栈中。
>当用ApplicationContext去启动standard模式的Activity时会报错。非Activity类型的Context(如ApplicationContext)并没有所谓的任务栈。解决这个问题的方法是为待启动Activity指定FLAG_ACTIVITY_NEW_TASK标记位，这样启动的时候就会为它创建一个新的任务栈，这个时候待启动Activity实际上是以singleTask模式启动的。

(2)singleTop:栈顶复用模式。在这种模式下，如果新Activity已经位于任务栈的栈顶，那么此Activity不会被重新创建，同时它的onNewIntent方法会被回调，onCreate、onStart不会被系统调用。
(3)singleTask:栈内复用模式。在这种模式下，只要Activity在一个栈中存在，那么多次启动此Activity都不会重新创建实例，和singleTop一样，系统也会回调onNewIntent。具体一点，当一个具有singleTask模式的Activity请求启动后，比如Activity A,**系统首先会寻找是否存在A想要的任务栈**，如果不存在，就重新创建一个任务栈，然后创建A的实例后把A放到栈中。
(4)singleInstance:单例模式。这是一种加强的singleTask模式，它除了具有singleTask模式的所有特性外，还加强了一点，就是具有此种模式的Activity只能单独地位于一个任务栈中。

假设目前有2个任务栈，前台任务栈的情况为AB,而后台任务栈的情况为CD,这里假设CD的启动模式均为singleTask。现在请求启动D,那么整个后台任务栈都会被切换到前台，整个时候整个后退列表变成了ABCD。当用户按back键的时候，列表中的Activity会一一出栈。
![](https://github.com/wslaimin/blog/raw/develop/pics/task.jpg)
什么是Activity所需要的任务栈？TaskAffinity(任务相关性,值为字符串且必须含有包名分隔符“.”)，这个参数标识了一个Activity所需要的任务栈的名字，默认情况下，所有Activity所需的任务栈的名字为应用的包名。TaskAffinity属性主要和singleTask启动模式或者allowTaskReparenting属性配对使用，其他情况下没有意义。

当TaskAffinity和allowTaskReparenting结合的时候。当一个应用A启动了应用B的某个Activity后，如果这个Activity的allowTaskReparenting属性为true,那么应用B被启动后，此Activity会直接从应用A的任务栈转移到应用B的任务栈。如果此Activity不是MAIN Activity出栈后会创建MAIN Activity。 

给Activity指定启动模式有两种方法：
1. 通过AndroidMenifest为Activity制动启动模式
2. 通过Intent中设置标志位来指定启动模式
两者之间是有区别的。首先，优先级上，第二种方式的优先级要高于第一种；在限定范围上有所不同，第一种无法直接为Activity设定FLAG_ACTIVITY_CLEAR_TOP标识，第二种无法为Activity指定singleInstance模式。

Activity的Flags:
1.FLAG_ACTIVITY_NEW_TASK，为Activity指定“singleTask”启动模式
2.FLAG_ACTIVITY_SINGLE_TOP，为Activity指定“singleTop”启动模式
3.FLAG_ACTIVITY_CLEAR_TOP,具有此标记位的Activity，当它启动时，在同一个任务栈中所有位于它上面的Activity都要出栈。和singleTask启动模式一起出现，被启动Activity的实例已经存在，系统会调用它的onNewIntent。被启动Activity采用standard启动模式，它连同它之上的Activity都要出栈，系统会创建新的Activity实例并放入栈顶。
4.FLAG_ACTIVITY_EXCLUDE_FROM_RECENTS，具有这个标记位的Activity不会出现在历史Activity的列表中。等同于在XML中指定android:excludeFromRecents="true"。 
##IntentFilter的匹配规则
启动Activity分为显示调用和隐式调用。二者共存的话以显示调用为主。IntentFilter中的过滤信息有action,category,data。

为了匹配过滤列表，需要同时匹配过滤列表中的action,category,data信息，否则匹配失败。一个Activity钟可以有多个intent-filter。

1.action的匹配规则
只要Intent中的action能够和过滤规则中的任何一个action相同即可匹配成功。
2.category的匹配规则
category是一个字符串，系统预定义了一些，也可以自己定义。Intent中如果含有category，那么所有的category都必须和过滤规则中的其中一个category相同。系统在调用startActivity或者startActiviForResult的时候会默认为Intent加上“android.intent.category.DEFAULT”这个category。
3.data的匹配规则
如果过滤规则中定义了data,那么Intent中必须也要定义可匹配的data。
data由两部分组成，mimeType和URI。
URI的结构：
```xml
<scheme>://<host>:<port>/[<path>|<pathPrefix>|<pathPattern>]
```
Scheme:URI的模式，如果URI中没有指定scheme，那么整个URI的其他参数无效。
Host:URI的主机名，如果host未指定，那么整个URI中的其他参数无效。
Port：URI的端口号，仅当URI中指定了scheme和host参数时port参数才有意义。
path,pathPattern和pathPrefix:path表示完整的路径信息；pathPattern也表示完整的路径信息，但是它里面可以包含通配符“*”，“*”表示0个或多个任意字符，需要注意的是，由于正则表达式的规范，如果想表示真实的字符串，那么“*”要写成“\\*”，“\”要写成“\\\\”(不是“\\”吗?);pathPrefix表示路径的前缀信息。

data也包括mimeType。
```xml
<intent-fileter>
    <data android:mimeType="image/*"/>
</inte>
```
要匹配上述data，那么Intent中的mimeType属性必须为“image/*”才能匹配，这种情况下没有指定URI，但却有默认值，URI的默认值为content和file。也就是说，URI部分的scheme必须为content或者file才能匹配。
要为Intent指定完整的data，必须要setDataAndType方法，不能先调用setData再这两个方法会彼此清除对方的值。

判断是否有Activity隐式Intent的两种方法：采用PakageManager的resolveActivity方法或者Intent的resolveActivity方法，如果它们找不到匹配的Activity就会返回null。
方法原型：
```java
public abstract List<ResolveInfo> queryIntentActivies(Intent intent,int flags);
public abstract ResolveInfo resolveActivity(Intent intent,int flags);
```
我们使用 MATCH_DEFAULT_ONLY标记位，匹配那些在intent-filter中声明了<category android:name="android.intent.category.DEFAULT">这个category的Activity。不使用这个标记位，可以把intent-filter中category不含DEFAULT的那些Activity给匹配出来，从而导致startActivit因为不含有DEFAULT的Activity是无法接收隐式Intent的。 
