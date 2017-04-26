#Taking Photos Simply笔记
以下内容均是使用已经存在的carmera app进行拍照。
##Request Camera Permission
如果拍照是你的app一个非常重要的功能，在manifest文件使用`<uses-feature>`标签，Google Play只会对有相机的设备可见。

```xml
<manifest ... >
    <uses-feature android:name="android.hardware.camera"
                  android:required="true" />
    ...
</manifest>
```

也可以在运行时调用hasSystemFeature(PackageManger.FEATURE_CAMERA)方法来检查相机是否可用。

##Take a Phote with the Camera App
调用camera app过程有三部分工作：实例化Intent对象、开启外部Activity、处理图片结果。
使用Intent开启拍照功能示例:

```java
static final int REQUEST_IMAGE_CAPTURE = 1;

private void dispatchTakePictureIntent() {
    Intent takePictureIntent = new Intent(MediaStore.ACTION_IMAGE_CAPTURE);
    if (takePictureIntent.resolveActivity(getPackageManager()) != null) {
        startActivityForResult(takePictureIntent, REQUEST_IMAGE_CAPTURE);
    }
}
```

##Get the Thumbnail
Android Camera app 返回的Intent，key为"data"，value为拍摄图片的缩略图。
代码示例如下:

```java
@Override
protected void onActivityResult(int requestCode, int resultCode, Intent data) {
    if (requestCode == REQUEST_IMAGE_CAPTURE && resultCode == RESULT_OK) {
        Bundle extras = data.getExtras();
        Bitmap imageBitmap = (Bitmap) extras.get("data");
        mImageView.setImageBitmap(imageBitmap);
    }
}
```

##Save the Full-size Photo
如果指定一个文件Android Camera app会保存全尺寸图片。
getExternalStoragepublicDirectory(Environment.DIRECTORY_PICTURE)返回的目录是存放可分享照片的合适地方。需要外部存储的写权限:

```xml
<manifest ...>
    <uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE" />
    ...
</manifest>
```

如果想照片只能你的app使用，可以存放在getExternalFilesDir()返回的目录。Android 4.4和之后版本写这个目录不需要WRITE_EXTERNAL_STORAGE权限。权限声明可以添加maxSdkVersion来指定需要权限的最大版本:

```xml
<manifest ...>
    <uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE"
                     android:maxSdkVersion="18" />
    ...
</manifest>
```

确定好文件目录，创建用时间戳命名的文件:

```java
String mCurrentPhotoPath;

private File createImageFile() throws IOException {
    // Create an image file name
    String timeStamp = new SimpleDateFormat("yyyyMMdd_HHmmss").format(new Date());
    String imageFileName = "JPEG_" + timeStamp + "_";
    File storageDir = getExternalFilesDir(Environment.DIRECTORY_PICTURES);
    File image = File.createTempFile(
        imageFileName,  /* prefix */
        ".jpg",         /* suffix */
        storageDir      /* directory */
    );

    // Save a file: path for use with ACTION_VIEW intents
    mCurrentPhotoPath = image.getAbsolutePath();
    return image;
}
```

为相片创建好文件后，开始创建Intent:

```java
static final int REQUEST_TAKE_PHOTO = 1;

private void dispatchTakePictureIntent() {
    Intent takePictureIntent = new Intent(MediaStore.ACTION_IMAGE_CAPTURE);
    // Ensure that there's a camera activity to handle the intent
    if (takePictureIntent.resolveActivity(getPackageManager()) != null) {
        // Create the File where the photo should go
        File photoFile = null;
        try {
            photoFile = createImageFile();
        } catch (IOException ex) {
            // Error occurred while creating the File
            ...
        }
        // Continue only if the File was successfully created
        if (photoFile != null) {
            Uri photoURI = FileProvider.getUriForFile(this,
                "com.example.android.fileprovider",photoFile);
            takePictureIntent.putExtra(MediaStore.EXTRA_OUTPUT, photoURI);
            startActivityForResult(takePictureIntent, REQUEST_TAKE_PHOTO);
        }
    }
}
```

>注意:getUriForFile(Context,String,File)返回content://URI。app target是Android 7.0(API level 24)或更高，跨包传递file://URI会抛出FileUriExposedException。在低版本系統中(測試HTC 4.4.3)使用content://URI不能保存拍照圖片。

在manifest文件配置FileProvider:

```xml
<application>
   ...
   <provider
        android:name="android.support.v4.content.FileProvider"
        android:authorities="com.example.android.fileprovider"
        android:exported="false"
        android:grantUriPermissions="true">
        <meta-data
            android:name="android.support.FILE_PROVIDER_PATHS"
            android:resource="@xml/file_paths"></meta-data>
    </provider>
    ...
</application>
```

res/xml/file_path.xml配置可选的文件路径：

```java
<?xml version="1.0" encoding="utf-8"?>
<paths xmlns:android="http://schemas.android.com/apk/res/android">
    <external-path name="my_images" path="Android/data/com.example.package.name/files/Pictures" />
</paths>
```

##Add the Photo to a Gallery

>注意：如果照片保存在getExternalFilesDir()返回的目录下，media scanner不能访问文件，因为在你的app私有目录下。

添加你的照到Media Provider's database:

```java
private void galleryAddPic() {
    Intent mediaScanIntent = new Intent(Intent.ACTION_MEDIA_SCANNER_SCAN_FILE);
    File f = new File(mCurrentPhotoPath);
    Uri contentUri = Uri.fromFile(f);
    mediaScanIntent.setData(contentUri);
    this.sendBroadcast(mediaScanIntent);
}
```

如果照片在你的app私有目录下，上面代码没有任何作用，使用FileProvider生成URI依然如此。例如：

```java
Intent intent=new Intent(Intent.ACTION_MEDIA_SCANNER_SCAN_FILE);
String filePath=Environment.getExternalStorageDirectory()+"/Android/data/com.addcn.car8891/files/84481_20130116142820494200_1.jpg";
File file=new File(filePath);
Uri uri=Uri.fromFile(file);
intent.setData(uri);
sendBroadcast(intent);
```

##Decode a Scaled Image
显示全尺寸的图片很容易内存溢出，使用适应目标View大小的图片可以极大减少内存：

```java
private void setPic() {
    // Get the dimensions of the View
    int targetW = mImageView.getWidth();
    int targetH = mImageView.getHeight();

    // Get the dimensions of the bitmap
    BitmapFactory.Options bmOptions = new BitmapFactory.Options();
    bmOptions.inJustDecodeBounds = true;
    BitmapFactory.decodeFile(mCurrentPhotoPath, bmOptions);
    int photoW = bmOptions.outWidth;
    int photoH = bmOptions.outHeight;

    // Determine how much to scale down the image
    int scaleFactor = Math.min(photoW/targetW, photoH/targetH);

    // Decode the image file into a Bitmap sized to fill the View
    bmOptions.inJustDecodeBounds = false;
    //inSampleSize取样数，表示inSampleSize个像素取一个像素
    bmOptions.inSampleSize = scaleFactor;
    bmOptions.inPurgeable = true;

    Bitmap bitmap = BitmapFactory.decodeFile(mCurrentPhotoPath, bmOptions);
    mImageView.setImageBitmap(bitmap);
}
```



