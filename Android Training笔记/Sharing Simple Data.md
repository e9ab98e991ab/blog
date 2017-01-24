#Sharing Simple Data
通过Intent来传递数据。
##Sending Simple Data to Other Apps
向多媒体添加文件：

```java
MediaScannerConnection.scanFile(this, new String[]{Environment.getExternalStorageDirectory() + "/" + "Download/20150525_091043.jpg"}, null,
                new MediaScannerConnection.OnScanCompletedListener() {
                    @Override
                    public void onScanCompleted(String path, Uri uri) {
                        System.out.println("path:"+path);
                        System.out.println("uri:"+uri);
                    }
});
```

调用sancFile()方法后可以在图库中看到Download文件夹。

##Adding an Easy Share Action
Android 4.0(API Level 14)或之后的版本，可以使用ShareActionProvider。低版本可以使用android.support.v7.widget.ShareActionProvider。

menu.xml:

```xml
<menu xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:support="http://schemas.android.com/apk/res-auto">
    <item
        android:id="@+id/share"
        support:showAsAction="always"
        android:title="Share"
        support:actionProviderClass="android.support.v7.widget.ShareActionProvider"/>
</menu>
```

设置分享Intent:

```java
  @Override
public boolean onCreateOptionsMenu(Menu menu) {
    getMenuInflater().inflate(R.menu.menu,menu);
    MenuItem item=menu.findItem(R.id.share);
    mShareActionProvider=(ShareActionProvider) MenuItemCompat.getActionProvider(item);
    return super.onCreateOptionsMenu(menu);
}

// Call to update the share intent
private void setShareIntent(Intent shareIntent) {
    if (mShareActionProvider != null) {
        mShareActionProvider.setShareIntent(shareIntent);
}
```




