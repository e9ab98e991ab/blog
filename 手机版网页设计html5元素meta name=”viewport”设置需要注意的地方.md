# 手机版网页设计html5元素meta name=”viewport”设置需要注意的地方

##viewport设置的详细参数

```html
<meta name="viewport"
 	    content="
 	        height = [pixel_value | device-height] ,
 	        width = [pixel_value | device-width ] ,
 	        initial-scale = float_value ,
 	        minimum-scale = float_value ,
 	        maximum-scale = float_value ,
 	        user-scalable = [yes | no] ,
 	        target-densitydpi = [dpi_value | device-dpi | high-dpi | medium-dpi | low-dpi]
 	    "
/>
```

target-densitydpi:一个屏幕像素密度是由屏幕分辨率决定的，通常定义为每英寸点的数量（dpi）。Android支持三种屏幕像素密度：低像素密度，中像素密度，高像素密度。一个低像素密度的屏幕每英寸上的像素点更少，而一个高像素密度的屏幕每英寸上的像素点更多。

>Android Browser和WebView默认屏幕为中像素密度。

下面是 target-densitydpi 属性的取值范围:

device-dpi –使用设备原本的 dpi 作为目标 dp。 不会发生默认缩放。
high-dpi – 使用hdpi 作为目标 dpi。 中等像素密度和低像素密度设备相应缩小。
medium-dpi – 使用mdpi作为目标 dpi。 高像素密度设备相应放大， 像素密度设备相应缩小。 这是默认的target density。
low-dpi -使用mdpi作为目标。 dpi。中等像素密度和高像素密度设备相应放大。
`<value>` – 指定一个具体的dpi 值作为target dpi。 这个值的范围必须在70–400之间。

为了防止Android Browser和WebView 根据不同屏幕的像素密度对你的页面进行缩放，你可以将viewport的target-densitydpi 设置为 device-dpi。当你这么做了，页面将不会缩放。相反，页面会根据当前屏幕的像素密度进行展示。在这种情形下，你还需要将viewport的width定义为与设备的width匹配，这样你的页面就可以和屏幕相适应。

initial-scale：初始缩放。即页面初始缩放程度。这是一个浮点值，是页面大小的一个乘数。例如，如果你设置初始缩放为“1.0”，那么，web页面在展现的时候就会以target density分辨率的1:1来展现。如果你设置为“2.0”，那么这个页面就会放大为2倍。

maximum-scale：最大缩放。即允许的最大缩放程度。这也是一个浮点值，用以指出页面大小与屏幕大小相比的最大乘数。例如，如果你将这个值设置为“2.0”，那么这个页面与target size相比，最多能放大2倍。

user-scalable：用户调整缩放。即用户是否能改变页面缩放程度。如果设置为yes则是允许用户对其进行改变，反之为no。默认值是yes。如果你将其设置为no，那么minimum-scale 和 maximum-scale都将被忽略，因为根本不可能缩放。

>所有的缩放值都必须在0.01–10的范围之内。

##实例一

```html
<head>
....
<meta name="viewport" content="width=device-width; initial-scale=1.0; maximum-scale=1.0; user-scalable=yes;">
...
</head>
```

解读：
width=device-width ：自适应屏幕大小
initial-scale=1.0 ：初始缩放是1比1
maximum-scale=1.0 ：最大允许缩放也是1比1
user-scalable=yes ：是否可缩放

好处：在大屏幕的手机上，用户无法缩放，控制了由于误操作导致的网页变形，保证了原始的网页设计。

坏处：这是我经常去逛的一个手机网站，我用的3.5寸的屌丝手机，很多小字与图片无法放大，根本不知道里面是什么内容，非常扯蛋。

结论：网页是给人看的，第一位重要的还是内容的传达，如果为了设计而损失了用户的真实使用，那是扯淡的设计。

##实例二

```html
<meta name="viewport" content="width=device-width, minimum-scale=1.0, maximum-scale=2.0" />
```

这是大站，新浪 sina.cn 的设置，这样用户至少可以缩放到2倍大小，基本满足大多数的需求。

##非自适应网页的研究

如果我们的网站没有做自适应的设置，让用户浏览完整版本的网页（有些网页没有必要去画蛇添足），如果这个时候我们在网页里面加入上面实例二代码，就会出现一个问题。

问题是：我们在手机端打开正常网页的时候（浏览器非自适应设置），网页会被自动缩放到1:1的状态，一般网页的宽度都超过了手机屏幕，而不能看见网页的全貌，这样有好也有坏。为了增强用户对网站的整体认知，我们还是建议缩放显示全貌比较好。

建议：在非自适应网页上面，建议最好不要加meta name=”viewport”的代码，如果你一定要加，请加如下模式：

```html
<meta name="viewport" content="width=980, minimum-scale=1.0, maximum-scale=2.0" />
```

其中 width=980 可以改成你网页的最小宽度。
但是，当你把 width=980 这个数值，设置小于或者等于手机屏幕的时候，这个设置值一般会失效，而按照缩放等级显示。

##推荐的设置

最后，如果是有适应手机版本的网页本站推荐设置成：

```html
<meta name="viewport" content="width=device-width, minimum-scale=1.0, maximum-scale=2.0" />
```

如果不是适应手机的网页则不要加viewport设置，但是两种网页都推荐在css里面加上：

```html
-webkit-text-size-adjust:none;
min-width:320px;
```

1、有些客户喜欢把浏览器的字体调到超大，结果会让你的网页字与字之间重叠。
2、给个最小宽度，太小的屏幕也给出横向滚动了，不要一竖列排下去了。





