# 第12章 Bitmap的加载和Cache

Lru是Least Recently Used的缩写，即最近最少使用算法，这种算法的核心思想是：当缓存快满时，会淘汰近期最少使用的缓存目标。

如何高效地加载Bitmap呢？核心思想是采用BitmapFactoryOptions来加载所需尺寸的图片。通过BitmapFactory.Options可以按一定的采样率来加载缩小后的图片。

通过BitmapFactory.Options来缩放图片，主要是用到了它的inSampleSize参数，即采样率。采样率同时作用于宽/高，这将导致缩放后的图片大小以采样率的2次方形式递减，即缩放比例为1/(inSampleSize的2次方)。最新的官方文档指出，inSampleSize的取值应该总是为2的指数，如果外界传递给系统的inSampleSize不为2的指数，那么系统会向下取整并选择一个最接近的2的指数来代替。经过验证这个结论并非在所有的Android版本上都成立。

获取采样率的流程：

 1. 将BitampFactory.Options的inJustDecodeBounds参数设为true并加载图片
 2. 从BitmaFactory.Options中取出图片的原始宽高信息，他们对应于outWidth和outHeight参数
 3. 根据采样率的规则并结合目标View的所需大小计算出采样率inSampleSize
 4. 将BitmapFactory.Options的inJustDecodeBounds参数设为false,然后重新加载

inJustDecodeBounds参数，当此参数为true时，BitmapFactory只会解析图片的原始宽/高信息，并不会去真正地加载图片。BitmapFactory获取的图片宽/高信息和图片的位置以及程序运行的设备有关，比如同一张图片放在不同的drawable目录下或者程序运行在不同屏幕密度的设备上，这都可能导致BitmapFactory获取到不同的结果。

##LruCache

##DiskLruCache

有符号数转换为无符号数：

byte b=-3;

0xFF&b或b+256（2的8次方）
 
