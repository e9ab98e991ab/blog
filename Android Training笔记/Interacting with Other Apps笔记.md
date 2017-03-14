#Interacting with Other Apps笔记

验证是否有能够处理Intent的Activity的方法：

```java
PackageManager packageManager = getPackageManager();
List activities = packageManager.queryIntentActivities(intent,
        PackageManager.MATCH_DEFAULT_ONLY);
boolean isIntentSafe = activities.size() > 0;
```

```java
// Verify the intent will resolve to at least one activity
if (intent.resolveActivity(getPackageManager()) != null) {
    startActivity(chooser);
}
```

有多个Activity可以处理Intent，每次都需要用户进行选择，可以使用createChooser()来创建Intent。

```java
ntent intent = new Intent(Intent.ACTION_SEND);
...

// Always use string resources for UI text.
// This says something like "Share this photo with"
String title = getResources().getString(R.string.chooser_title);
// Create intent to show chooser
Intent chooser = Intent.createChooser(intent, title);

// Verify the intent will resolve to at least one activity
if (intent.resolveActivity(getPackageManager()) != null) {
    startActivity(chooser);
}
```

如果想要Activity返回值，调用setResult()方法，之后再调用finish()方法关闭Activity。

按下返回键，返回的结果默认设置为RESULT_CANCELED。




