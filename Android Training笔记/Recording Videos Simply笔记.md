#Recording Videos Simply笔记
##Request Camera Permission
声明你的app需要相机:

```xml
<manifest ... >
    <uses-feature android:name="android.hardware.camera"
                  android:required="true" />
    ...
</manifest>
```

也可以在运行时，调用hasSystemFeature(PackageManager.FEATURE_CAMERA)检查相机是否可用。
##Record a Video with a Camera App
代码示例：

```java
static final int REQUEST_VIDEO_CAPTURE = 1;

private void dispatchTakeVideoIntent() {
    Intent takeVideoIntent = new Intent(MediaStore.ACTION_VIDEO_CAPTURE);
    if (takeVideoIntent.resolveActivity(getPackageManager()) != null) {
        startActivityForResult(takeVideoIntent, REQUEST_VIDEO_CAPTURE);
    }
}
```

和拍照一样，给intent添加key为MediaStore.EXTRA_OUTPUT，value为文件URI可以指定拍摄视频的文件位置。

##View the Video

```java
@Override
protected void onActivityResult(int requestCode, int resultCode, Intent intent) {
    if (requestCode == REQUEST_VIDEO_CAPTURE && resultCode == RESULT_OK) {
        Uri videoUri = intent.getData();
        mVideoView.setVideoURI(videoUri);
    }
}
```

返回的是content://URI。

>注意：用模拟器运行，可能会有返回intent为空的情况，最好用真机测试。
