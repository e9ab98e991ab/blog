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

###代码示例
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

##Sharing File
###Receive File Requests
为了从客户端app接收文件请求，并且用一个content URI作为相应，你的app需要一个提供文件选择的Activity。客户端app用带ACTION_PICK的action，调用startActivityForResult()方法。
###Create a File Selection Activity

```java
<manifest xmlns:android="http://schemas.android.com/apk/res/android">
    ...
        <application>
        ...
            <activity
                android:name=".FileSelectActivity"
                android:label="@File Selector" >
                <intent-filter>
                    <action
                        android:name="android.intent.action.PICK"/>
                    <category
                        android:name="android.intent.category.DEFAULT"/>
                    <category
                        android:name="android.intent.category.OPENABLE"/>
                    <data android:mimeType="text/plain"/>
                    <data android:mimeType="image/*"/>
                </intent-filter>
            </activity>
```

android.intent.category.OPENABLE表示intent的URI能够被openFileDescriptor(Uri,String)方法开启。

##Define the file selection Activity in code

```java
public class MainActivity extends Activity {
    // The path to the root of this app's internal storage
    private File mPrivateRootDir;
    // The path to the "images" subdirectory
    private File mImagesDir;
    // Array of files in the images subdirectory
    File[] mImageFiles;
    // Array of filenames corresponding to mImageFiles
    String[] mImageFilenames;
    // Initialize the Activity
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        ...
        // Set up an Intent to send back to apps that request a file
        mResultIntent =
                new Intent("com.example.myapp.ACTION_RETURN_FILE");
        // Get the files/ subdirectory of internal storage
        mPrivateRootDir = getFilesDir();
        // Get the files/images subdirectory;
        mImagesDir = new File(mPrivateRootDir, "images");
        // Get the files in the images subdirectory
        mImageFiles = mImagesDir.listFiles();
        // Set the Activity's result to null to begin with
        setResult(Activity.RESULT_CANCELED, null);
        /*
         * Display the file names in the ListView mFileListView.
         * Back the ListView with the array mImageFilenames, which
         * you can create by iterating through mImageFiles and
         * calling File.getAbsolutePath() for each File
         */
         ...
    }
    ...
}
```

##Respond to a File Selection
URI权限不能用于Uri.fromFile()方法生成的file://URis，能用于与Content Providers相关的URIs。FileProvider API可以帮助创建这种URIs。

实例代码：

```java
protected void onCreate(Bundle savedInstanceState) {
    ...
    // Define a listener that responds to clicks on a file in the ListView
    mFileListView.setOnItemClickListener(
        new AdapterView.OnItemClickListener() {
        @Override
        /*
        * When a filename in the ListView is clicked, get its
         * content URI and send it to the requesting app
         */
        public void onItemClick(AdapterView<?> adapterView,
        View view,int position,long rowId) {
            /*
             * Get a File for the selected file name.
             * Assume that the file names are in the
             * mImageFilename array.
             */
            File requestFile = new File(mImageFilename[position]);
            /*
             * Most file-related method calls need to be in
             * try-catch blocks.
             */
            // Use the FileProvider to get a content URI
            try {
                fileUri = FileProvider.getUriForFile(
                    MainActivity.this,"com.example.myapp.fileprovider",
                        requestFile);
                } catch (IllegalArgumentException e) {
                    Log.e("File Selector",
                          "The selected file can't be shared: " +
                          clickedFilename);
                }
                ...
        }
    });
        ...
}
```

##Grant Permission for the File
要使content URI能够访问或修改数据，还必须对客户端app进行授权。授权的方式有：

 - 在`<provider/>`里通过android:grantUriPermissions、android:readPermission、android:writePermission属性，或`<grant-uri-permission>`标签进行授权。
 - Intent设置了URI,比如调用setData(Uri)，调用Intent的addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION)进行授权。接收app的任务栈结束时，权限自动撤销。
 - 通过Context.grantUriPermission()方法给content URI授权。有个缺点是撤销权限需要调用Context.revokeUriPermission()方法。

##Share the File with the Requesting App
代码片段如下：

```java
protected void onCreate(Bundle savedInstanceState) {
    ...
    // Define a listener that responds to clicks on a file in the ListView
    mFileListView.setOnItemClickListener(
        new AdapterView.OnItemClickListener() {
        @Override
        public void onItemClick(AdapterView<?> adapterView,
            View view,int position,long rowId) {
                ...
                if (fileUri != null) {
                    ...
                    // Put the Uri and MIME type in the result Intent
                    mResultIntent.setDataAndType(
                            fileUri,
                            getContentResolver().getType(fileUri));
                    // Set the result
                    MainActivity.this.setResult(Activity.RESULT_OK,
                            mResultIntent);
                    } else {
                        mResultIntent.setDataAndType(null, "");
                        MainActivity.this.setResult(RESULT_CANCELED,
                                mResultIntent);
                    }
                }
    });
```

##Requesting a Shared File
代码示例：

```java
public class MainActivity extends Activity {
    private Intent mRequestFileIntent;
    private ParcelFileDescriptor mInputPFD;
    ...
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        mRequestFileIntent = new Intent(Intent.ACTION_PICK);
        mRequestFileIntent.setType("image/jpg");
        ...
    }
    ...
    protected void requestFile() {
        /**
         * When the user requests a file, send an Intent to the
         * server app.
         * files.
         */
            startActivityForResult(mRequestFileIntent, 0);
        ...
    }
    ...
}
```

##Access the Requested File 
代码示例：

```java
/*
     * When the Activity of the app that hosts files sets a result and calls
     * finish(), this method is invoked. The returned Intent contains the
     * content URI of a selected file. The result code indicates if the
     * selection worked or not.
     */
    @Override
    public void onActivityResult(int requestCode, int resultCode,
            Intent returnIntent) {
        // If the selection didn't work
        if (resultCode != RESULT_OK) {
            // Exit without doing anything else
            return;
        } else {
            // Get the file's content URI from the incoming Intent
            Uri returnUri = returnIntent.getData();
            /*
             * Try to open the file for "read" access using the
             * returned URI. If the file isn't found, write to the
             * error log and return.
             */
            try {
                /*
                 * Get the content resolver instance for this context, and use it
                 * to get a ParcelFileDescriptor for the file.
                 */
                mInputPFD = getContentResolver().openFileDescriptor(returnUri, "r");
            } catch (FileNotFoundException e) {
                e.printStackTrace();
                Log.e("MainActivity", "File not found.");
                return;
            }
            // Get a regular file descriptor for the file
            FileDescriptor fd = mInputPFD.getFileDescriptor();
            ...
        }
    }
```

openFileDescriptor()方法的第二个参数是使用的文件模式，r表示读,w表示写，rw表示读写。

##Retrieving File Information
###Retrieve a File's MIME Type
代码示例：

```java
...
/*
 * Get the file's content URI from the incoming Intent, then
 * get the file's MIME type
 */
Uri returnUri = returnIntent.getData();
String mimeType = getContentResolver().getType(returnUri);
...
```

###Retrieve a File's Name and Size
FileProvider类默认实现的query()方法返回的Cursor有连个列名：

 - DISPLAY_NAME 文件名。
 - SIZE 文件字节数。

示例代码：

```java
...
/*
 * Get the file's content URI from the incoming Intent,
 * then query the server app to get the file's display name
 * and size.
 */
Uri returnUri = returnIntent.getData();
Cursor returnCursor =
    getContentResolver().query(returnUri, null, null, null, null);
/*
 * Get the column indexes of the data in the Cursor,
 * move to the first row in the Cursor, get the data,
 * and display it.
 */
int nameIndex = returnCursor.getColumnIndex(OpenableColumns.DISPLAY_NAME);
int sizeIndex = returnCursor.getColumnIndex(OpenableColumns.SIZE);
returnCursor.moveToFirst();
TextView nameView = (TextView) findViewById(R.id.filename_text);
TextView sizeView = (TextView) findViewById(R.id.filesize_text);
nameView.setText(returnCursor.getString(nameIndex));
sizeView.setText(Long.toString(returnCursor.getLong(sizeIndex)));
...
```
 
 
