# 记一次Android OOM探险之旅

## 分析利器

### 查看内存状态

adb shell dumpsys meminfo packageName

### Dump Java Heap

用Android Studio获取Java Heap文件

![](https://www.github.com/wslaimin/blog/raw/master/pics/dump_heap.png)

用hprof-conv命令转化文件，转化后的文件可以用MAT打开。 

### MAT使用

[MAT(Memory Analyzer Tool)](http://www.eclipse.org/mat/downloads.php)

1. Dominator Tree:列出存活的对象

![](https://www.github.com/wslaimin/blog/raw/master/pics/dominator_tree.png)

2. 右键对象 -> Path To GC Roots -> exclude all phantom/weak/soft etc.references

![](https://www.github.com/wslaimin/blog/raw/master/pics/gc.png)

3. 根据对象的被引用关系，分析内存泄漏原因

![](https://www.github.com/wslaimin/blog/raw/master/pics/gc_path.png)

### GIMP查看图片

借助GIMP能够查看MAT导出的图片。

1. 导出图片字节码，保存为.data扩展名

![](https://www.github.com/wslaimin/blog/raw/master/pics/export_data.png)

2. 查看图片大小

![](https://www.github.com/wslaimin/blog/raw/master/pics/pic_size.png)

3. 用GIMP打开*.data，图片类型选择RGB Alpha，填上宽度、高度

![](https://www.github.com/wslaimin/blog/raw/master/pics/gimp.png)

## 内存泄漏分析

上面Bitmap的GC Path可以看出：loadResultListener引用的对象是匿名内部类对象，持有外部Activity引用。这个对象被google gms引用，因为google gms的声明周期远超Activity，所以导致Bitmap不能被回收，甚至连Activity都不能被回收。

Ps:[为什么Handler会导致内存泄漏](https://www.androiddesignpatterns.com/2013/01/inner-class-handler-memory-leak.html)
