#Android 的坐标系及矩阵变换

##Android的坐标系
##2D坐标系
android的2d坐标系如图所示，水平向右为X轴正方向，竖直向下为Y轴正方向，原点为屏幕左上角。
![](https://www.github.com/wslaimin/blog/raw/master/pics/xy.jpg)

><font color="0xff000000">注意：以屏幕左上角为原点的坐标系，称作绝对坐标系，将原点平移到View的左上角，称作相对坐标系。对View进行操作时，更多使用的是相对坐标系。</font>

##3D坐标系
###左手坐标系
在计算机科学中，大多3D坐标系使用的是左手坐标系(包括Android)，所以，在这里只介绍左手坐标系。

用左手确定3D坐标系：
![](https://www.github.com/wslaimin/blog/raw/master/pics/zuoshou.PNG)

在Android设备中反应出的3D坐标系是，X轴水平向右，Y轴竖直向上，Z轴垂直屏幕指向屏幕内部。

有没有觉得很疑惑，明明在2D坐标系中Y轴正方向是竖直向下的，怎么到3D坐标系就变成竖直向上了？其实，2D坐标系和3D坐标系是没有直接关系的，绘制View使用的是2D坐标系，3D坐标系则是进行3D变换，求得3D变换矩阵。2D坐标通过3D变换矩阵，改变x坐标和y坐标。

><font color="0xff000000">注意：2D和3D坐标系之间没有直接关系</font>

同样，根据坐标原点的不同也可分为绝对坐标系和相对坐标系。

###确定旋转正方向
由于使用的是左手坐标系，伸出左手，握住坐标系，大拇指指向轴的正方向，四指指向的方向即为旋转的正方向。如图所示：
![](https://www.github.com/wslaimin/blog/raw/master/pics/fangxiang.png)

###屏幕上点的表示
在屏幕上显示的点，不仅仅有x，y坐标(这里的x，y坐标是2D坐标系的坐标)，其实还有z轴的影响，z轴坐标可以理解成物体与镜头的距离。所以这里对应的像素描述由一个3行一列的矩阵来表示：
![](https://www.github.com/wslaimin/blog/raw/master/pics/xy1.png)
x，y分别代表x，y轴上的坐标，而1代表屏幕在z轴上的坐标为默认的。如果将1变大，那么屏幕会拉远， 图形会变小。

###旋转变化的是坐标系
任何变换都是基于坐标系变化发生的。比如，绕Z轴旋转，改变的是点在XOY平面的映射，所以绕Z轴旋转等同于XOY的坐标系旋转后，旧坐标系的点在新坐标系中的坐标的计算过程。计算过程如下：

在原坐标系xoy中,  绕原点沿逆时针方向旋转θ度， 变成座标系 sot。
设有某点p，在原坐标系中的坐标为 (x, y), 旋转后的新坐标为(s, t)。
![](https://www.github.com/wslaimin/blog/raw/master/pics/xuanzhuan.gif)

oa = y sin(θ)   (2.1)
as = x cos(θ)   (2.2)
综合(2.1)，(2.2) 2式
s =  os = oa + as = x cos(θ) + y sin(θ) 
t =  ot = ay – ab = y cos(θ) – x sin(θ)

用行列式表达如下：
![](https://www.github.com/wslaimin/blog/raw/master/pics/zxuanzhuan.png)

由上面的结果可以得出绕Z轴旋转的变换矩阵为：
![](https://www.github.com/wslaimin/blog/raw/master/pics/zbianhuan.png)

更多3D旋转矩阵可参考：
http://blog.csdn.net/zsq306650083/article/details/8773996

###变换矩阵在2D平面的表现
根据绕Z轴旋转的变换矩阵，可以求得旋转后的坐标。θ为绕Z轴旋转角度，P0(x0,y0，1)为旋转前的坐标，P1(x1,y1，1)为旋转后的坐标。
计算出：
![](https://www.github.com/wslaimin/blog/raw/master/pics/x1y1.png)
由此可以得到在XOY平面的旋转示意图：
![](https://www.github.com/wslaimin/blog/raw/master/pics/xyxuanzhuan.png)

<font color="0xff000000">可以看到，变换矩阵不仅决定点变换后的坐标，也决定了点旋转的方向。</font>

###矩阵的初等变换
上面的矩阵相乘用到的是矩阵初等变换的知识，这里贴一下矩阵初等变换的一些结论：
![](https://www.github.com/wslaimin/blog/raw/master/pics/jzjielun.png)

###改变旋转的中心点
以上得出的绕Z轴旋转的旋转矩阵是基于原点，如果要改变旋转的中心点，该怎么做？
设中心点坐标O1(x2,y2,1),P0(x0,y0,1),XO1Y坐标系中P1(x1,y1,1),变换后P3(x3,y3,1)。

经过旋转矩阵变换P3的坐标：
![](https://www.github.com/wslaimin/blog/raw/master/pics/x3y3.png)

P0和P1之间的关系：
![](https://www.github.com/wslaimin/blog/raw/master/pics/guanxi.png)

P3的坐标:
![](https://www.github.com/wslaimin/blog/raw/master/pics/p3.png)

由于有：
![](https://www.github.com/wslaimin/blog/raw/master/pics/chudeng.png)

最终得到P3在XOY坐标系的坐标：
![](https://www.github.com/wslaimin/blog/raw/master/pics/jieguo.png)

上面是纯数学计算过程，其实通过矩阵的初等变换来更好理解和记忆：
![](https://www.github.com/wslaimin/blog/raw/master/pics/koujue.png)

所以，在获得变换矩阵后，如果需要改变中心点坐标，通常会使用下面两行代码：

```java
matrix.preTranslate(-centerX,-centerY);
matrix.postTranslate(centerX,centerY);
```

###关于Camera类
为了方便获取变换矩阵，Android提供了Camera类获取变换矩阵，但是要注意，所有的变换都是基于原点的。

##绕Z轴旋转Demo
https://github.com/wslaimin/RotationZ.git

![](https://www.github.com/wslaimin/blog/raw/master/pics/zdemo.JPG)
#参考文章
http://www.2cto.com/kf/201605/510416.html
http://blog.csdn.net/zsq306650083/article/details/8773996
http://blog.csdn.net/Tangyongkang/article/details/5484636
http://blog.csdn.net/flash129/article/details/8234599


