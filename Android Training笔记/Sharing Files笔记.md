#Sharing Files笔记

##Setting Up File Sharing

为了安全地向别的app提供文件，使用URI是个很好的方法。FielProvider可以为文件生成URI。

指定FileProvider:

```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="com.example.myapp">
    <application
        ...>
        <provider
            android:name="android.support.v4.content.FileProvider"
            android:authorities="com.example.myapp.fileprovider"
            android:grantUriPermissions="true"
            android:exported="false">
            <meta-data
                android:name="android.support.FILE_PROVIDER_PATHS"
                android:resource="@xml/filepaths" />
        </provider>
        ...
    </application>
</manifest>
```

android:exported表示你的app声明的FileProvider能否被其他app使用。android:resource表示能够分享文件的目录。

指定可以分享的目录：
在res/xm目录下创建filepaths.xml，在文件中声明目录。`<files-path`>标签，指定内部存储的files目录；类似的还有`<external-path`>、`<cache-path`>标签。代码片段如下：

```xml
<paths>
    <files-path path="images/" name="myimages" />
</paths>
```

在使用`<external-cache-path>`时，会报错：Caused by: java.lang.IllegalArgumentException: Failed to find configured root that contains /storage/emulated/0/Android/data/com.addcn.car8891/cache/123.txt。这是什么原因？

>注意：XML文件是唯一指定分享目录的方式。

上诉例子，FileProvider返回的URI为content://com.example.myapp.fileprovider/myimages/default_image.jpg。

##代码示例
使用FileProvider将文件转化成Uri：

```java
Uri uri=FileProvider.getUriForFile(this,"com.addcn.car8891.fileprovider",new File(getExternalCacheDir(),"123.txt"));
    if(mShareActionProvider!=null){
        Intent intent=new Intent(Intent.ACTION_SEND);
        intent.setType("text/plain");
        intent.putExtra(Intent.EXTRA_STREAM,uri);
        mShareActionProvider.setShareIntent(intent);
    }
```

可分享文件目录声明：

```xml
<paths xmlns:android="http://schemas.android.com/apk/res/android">
    <!--<external-cache-path path="" name="haha"/>-->
    <external-path path="" name="heihei"/>
</paths>
```

