#Saving Data笔记
SharedPreferences APIs是用来读/写键值对的，Preference APIs是用来构建app设置页面的UI(其使用SharedPreference来保存app设置)。

##SharedPreferences
###获取SharedPreferences引用

 - 通过Context的getSharedPreference()方法。

```java
Context context = getActivity();
SharedPreferences sharedPref = context.getSharedPreferences(
        getString(R.string.preference_file_key), Context.MODE_PRIVATE);
```

 - 通过Activity的getPreferences()方法，会为Activity创建专属SharedPreferences。

```java
SharedPreferences sharedPref = getActivity().getPreferences(Context.MODE_PRIVATE);
```

 - 通过PreferenceManager的getDefaultSharedPreferences()方法，会为app创建专属SharedPreferences。

>注意：如果使用MODE_WORLD_READABLE或MODE_WORLD_WRITEABLE创建SharedPreferences文件，其他app知道文件名即可访问数据。
 
###向SharedPreferences写入数据

```java
SharedPreferences sharedPref = getActivity().getPreferences(Context.MODE_PRIVATE);
SharedPreferences.Editor editor = sharedPref.edit();
editor.putInt(getString(R.string.saved_high_score), newHighScore);
editor.commit();
```

###从SharedPreferences读取数据

```java
SharedPreferences sharedPref = getActivity().getPreferences(Context.MODE_PRIVATE);
int defaultValue = getResources().getInteger(R.string.saved_high_score_default);
long highScore = sharedPref.getInt(getString(R.string.saved_high_score), defaultValue);
```

##Saving Files
###内部存储和外部存储
所有Android设备都有两个文件存储："内部"和"外部"存储。有些设备能够挂载sd卡作为外部存储；有些不能挂载sd卡的设备会把存储分成内部和外部存储，这种外部存储是不能移除的。
有关手机存储的解释：
![](https://github.com/wslaimin/blog/raw/master/pics/storage.JPG)

>提示：尽管app默认安装在内部存储，可以在`<manifest>`标签通过android:installLocation属性选择安装位置。

###获取外部存储的权限

```java
<manifest ...>
    <uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE" />
    ...
</manifest>
```

>注意：当前，获取外部存储的写权限，同时也获得了读权限。这在之后的发布版本中可能会修改。最好明确指定 READ_EXTERNAL_STORAGE权限。Android 4.4和之后版本，getExternalFilesDir()方法返回的目录写入不需要WRITE_EXTERNAL_STORAGE权限。

###在内部存储上保存文件
获取内部存储目录的两种方法：
getFilesDir()
　　返回File代表app内部存储的目录。
getCacheDir()
　　返回File代表app内部存储的临时缓存目录。最好给缓存文件设置个阈值，删除不用的文件。系统在低存储的情况下坑你会删除缓存文件。
    
openFileOutput()
　　获取内部存储文件的输出流，文件不存在会创建文件。
　　
```java
String filename = "myfile";
String string = "Hello world!";
FileOutputStream outputStream;

try {
  outputStream = openFileOutput(filename, Context.MODE_PRIVATE);
  outputStream.write(string.getBytes());
  outputStream.close();
} catch (Exception e) {
  e.printStackTrace();
}
```

如果需要缓存文件，可以使用createTempFile()创建文件。

```java
public File getTempFile(Context context, String url) {
    File file;
    try {
        String fileName = Uri.parse(url).getLastPathSegment();
        file = File.createTempFile(fileName, null, context.getCacheDir());
    } catch (IOException e) {
        // Error while creating file
    }
    return file;
}
```

>注意：app在内部存储的文件夹名称是由包名决定的。如果没有显示的指定文件是可读或可写的，其他app不能访问这些文件。

###在外部存储保存文件
因为外部存储可能不可用——比如SD卡被移除，挂载到PC，因此，在使用外部存储前先调用getExternalStorageState()检查外部存储的状态。

```java
/* Checks if external storage is available to at least read */
public boolean isExternalStorageReadable() {
    String state = Environment.getExternalStorageState();
    if (Environment.MEDIA_MOUNTED.equals(state) ||
        Environment.MEDIA_MOUNTED_READ_ONLY.equals(state)) {
        return true;
    }
    return false;
}
```

外部存储分为两种类型：
Public files
　　所有用户和app都可以使用。通常用来保存照片和下载文件。
Private files
　　属于app的文件，卸载app文件会被删除。其他用户或app知道包名的情况下也可使用文件。
　　
不管使用getExternalStoragePublicDirectory()还是getExternalFilesDir()，尽量使用API提供的目录常量，比如DIRECTORY_PICTURE。这些目录名能保证系统正确处理文件。比如，存储在DIRECTORY_RINGTONES目录会被系统媒体管理认为是铃声而不是音乐。

###查询剩余空间
如果在存储数据前知道数据大小，可以调用getFreeSpace()或getTotalSpace查看空间大小。

系统并不保证可以写入和getFreeSpace()返回值一样的数据量。

>注意：在存储数据前不需要检查可用空间，可以捕获IOException来做存储数据失败的操作。

###删除文件
获取文件引用，调用delete()方法。

```java
myFile.delete();
```

如果文件存在内部存储，可以调用Context的deleteFile()方法。

```java
myContext.deleteFile(fileName);
```

>注意：当卸载app，android系统会删除的文件
 - 所有内部存储的文件。
 - 所有用getExternalFilesDir()存在外部存储文件。getExternalFilesDir()返回的目录为Android/data/data/your_package/
 
##在数据库保存数据
###定义Scheme和Contract
创建定义URIs、tables和columns常量的contract类，可以方便对这些常量进行统一管理。

>通过实现BaseColumns接口，内部类可以继承主见字段_ID，在使用cursor adaptors很有用。

定义table和colum如下：

```java
public final class FeedReaderContract {
    // To prevent someone from accidentally instantiating the contract class,
    // make the constructor private.
    private FeedReaderContract() {}

    /* Inner class that defines the table contents */
    public static class FeedEntry implements BaseColumns {
        public static final String TABLE_NAME = "entry";
        public static final String COLUMN_NAME_TITLE = "title";
        public static final String COLUMN_NAME_SUBTITLE = "subtitle";
    }
}
```

###使用SQL Helper创建数据库
创建和删除table的SQL：

```java
private static final String SQL_CREATE_ENTRIES =
    "CREATE TABLE " + FeedEntry.TABLE_NAME + " (" +
    FeedEntry._ID + " INTEGER PRIMARY KEY," +
    FeedEntry.COLUMN_NAME_TITLE + " TEXT," +
    FeedEntry.COLUMN_NAME_SUBTITLE + " TEXT)";

private static final String SQL_DELETE_ENTRIES =
    "DROP TABLE IF EXISTS " + FeedEntry.TABLE_NAME;
```

数据库存在内部存储，默认情况下其他应用不能使用。

>注意：getWritableDatabase()和getReadableDatabase()可能是耗时操作，最好不要在主线程调用。

继承SQLiteOpenHelper:

```java
public class FeedReaderDbHelper extends SQLiteOpenHelper {
    // If you change the database schema, you must increment the database version.
    public static final int DATABASE_VERSION = 1;
    public static final String DATABASE_NAME = "FeedReader.db";

    public FeedReaderDbHelper(Context context) {
        super(context, DATABASE_NAME, null, DATABASE_VERSION);
    }
    public void onCreate(SQLiteDatabase db) {
        db.execSQL(SQL_CREATE_ENTRIES);
    }
    public void onUpgrade(SQLiteDatabase db, int oldVersion, int newVersion) {
        // This database is only a cache for online data, so its upgrade policy is
        // to simply to discard the data and start over
        db.execSQL(SQL_DELETE_ENTRIES);
        onCreate(db);
    }
    public void onDowngrade(SQLiteDatabase db, int oldVersion, int newVersion) {
        onUpgrade(db, oldVersion, newVersion);
    }
}
```

实例化SQLiteOpenHelper子类:

```java
FeedReaderDbHelper mDbHelper = new FeedReaderDbHelper(getContext());
```
###向数据库插入数据
插入一条记录：

```java
// Gets the data repository in write mode
SQLiteDatabase db = mDbHelper.getWritableDatabase();

// Create a new map of values, where column names are the keys
ContentValues values = new ContentValues();
values.put(FeedEntry.COLUMN_NAME_TITLE, title);
values.put(FeedEntry.COLUMN_NAME_SUBTITLE, subtitle);

// Insert the new row, returning the primary key value of the new row
long newRowId = db.insert(FeedEntry.TABLE_NAME, null, values);
```

insert()的第二个参数表示，当ContentValues为空(没有放入values)，参数为空，不会向表插入记录；否则，插入一条记录参数指定的列为null。

###从数据库读取数据

```java
SQLiteDatabase db = mDbHelper.getReadableDatabase();

// Define a projection that specifies which columns from the database
// you will actually use after this query.
String[] projection = {
    FeedEntry._ID,
    FeedEntry.COLUMN_NAME_TITLE,
    FeedEntry.COLUMN_NAME_SUBTITLE
    };

// Filter results WHERE "title" = 'My Title'
String selection = FeedEntry.COLUMN_NAME_TITLE + " = ?";
String[] selectionArgs = { "My Title" };

// How you want the results sorted in the resulting Cursor
String sortOrder =
    FeedEntry.COLUMN_NAME_SUBTITLE + " DESC";

Cursor cursor = db.query(
    FeedEntry.TABLE_NAME,                     // The table to query
    projection,                               // The columns to return
    selection,                                // The columns for the WHERE clause
    selectionArgs,                            // The values for the WHERE clause
    null,                                     // don't group the rows
    null,                                     // don't filter by row groups
    sortOrder                                 // The sort order
    );
```

因为Cursor的初始位置为-1，所以在读取数据之前，要先调用move方法。用cursor读取数据：

```java
List itemIds = new ArrayList<>();
while(cursor.moveToNext()) {
  long itemId = cursor.getLong(
      cursor.getColumnIndexOrThrow(FeedEntry._ID));
  itemIds.add(itemId);
}
cursor.close();
```

###从数据库删除记录
selection格式分成selection语句和selection参数两部分，这样可以防止SQL注入。

```java
// Define 'where' part of query.
String selection = FeedEntry.COLUMN_NAME_TITLE + " LIKE ?";
// Specify arguments in placeholder order.
String[] selectionArgs = { "MyTitle" };
// Issue SQL statement.
db.delete(FeedEntry.TABLE_NAME, selection, selectionArgs);
```

###更新数据库

```java
SQLiteDatabase db = mDbHelper.getReadableDatabase();

// New value for one column
ContentValues values = new ContentValues();
values.put(FeedEntry.COLUMN_NAME_TITLE, title);

// Which row to update, based on the title
String selection = FeedEntry.COLUMN_NAME_TITLE + " LIKE ?";
String[] selectionArgs = { "MyTitle" };

int count = db.update(
    FeedReaderDbHelper.FeedEntry.TABLE_NAME,
    values,
    selection,
    selectionArgs);
```

###保持数据库连接
由于getWritableDatabase()和getReadableDatabase()在database关闭时调用开销很高，所以尽可能保持数据库连接 状态。在Activity的onDestroy()关闭数据库是最佳时机。