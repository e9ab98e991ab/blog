# ImageView支持Exif
##什么是Exif
可交换图像文件格式,简称为Exif（Exchangeable image file format）。是专门为数码相机的照片设定的，可以记录数码照片的属性信息和拍摄数据。

exif中包含图片的方向信息，这个是本文讨论的重点。<a href="http://sylvana.net/jpegcrop/exif_orientation.html">exif orientation详细介绍</a>

链接中的orientation value,0th row,0th column的关系换成下图更好理解：
![](https://www.github.com/wslaimin/blog/raw/master/pics/exif_orientation.png)

可以用手机拍摄一张图片，在<a href="http://metapicz.com/#landing">这个网站</a>查看exif。
##Camera Sensor方向
图像传感器(Image Sensor)，是有默认的取景方向的。Android Image Sensor取景方向如图所示：
![](https://www.github.com/wslaimin/blog/raw/master/pics/sensor.png)

Android Camera Sensor采集图像数据都是这个视角。所以，手机旋转90°拍摄的图片放在电脑上显示的图像并不是正的。

为什么手机相册显示的图片却是正的？
因为相册app有对图片进行处理。简单来说原理就是，获取exif中的orientation信息，然后将图片旋转相应角度。

##ImageView 支持exif orientation
获取图片exif，然后取得旋转角度，将图片旋转相应角度后给ImageView显示。

```java
private class DisplayTask implements Runnable {
    @Override
    public void run() {
        try {
            InputStream in = getAssets().open("pic.jpg");
            BitmapFactory.Options options = new BitmapFactory.Options();
            options.inJustDecodeBounds = true;
            BitmapFactory.decodeStream(getStream(in), null, options);
            options.inSampleSize = caculateInSampleSize(options, imageView.getMeasuredWidth(), imageView.getMeasuredHeight());
            options.inJustDecodeBounds = false;
            Bitmap bmp = BitmapFactory.decodeStream(getStream(in), null, options);

            int degree=0;
            switch (getImageOrientation(getStream(in))){
                case android.media.ExifInterface.ORIENTATION_ROTATE_90:
                    degree=90;
                    break;
                case android.media.ExifInterface.ORIENTATION_ROTATE_180:
                    degree=180;
                    break;
                case android.media.ExifInterface.ORIENTATION_ROTATE_270:
                    degree=270;
                    break;
            }
            //旋转图片
            Matrix matrix=new Matrix();
            matrix.postRotate(degree);
            Bitmap finalBmp=Bitmap.createBitmap(bmp,0,0,bmp.getWidth(),bmp.getHeight(),matrix,true);
            //居中
            matrix.reset();
            matrix.postTranslate((imageView.getMeasuredWidth()-finalBmp.getWidth())/2,(imageView.getMeasuredHeight()-finalBmp.getHeight())/2);
            imageView.setImageMatrix(matrix);
            imageView.setImageBitmap(finalBmp);
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

##完整代码
<a href="https://github.com/wslaimin/ExifOrientation">下载链接</a>


