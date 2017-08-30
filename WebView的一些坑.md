#WebView的一些坑
##WebViewClient和WebChromeClient区别
WebViewClient： It will be called when things happen that impact the rendering of the content
WebChromeClient：This class is called when something that might impact a browser UI happens

##WebView不现实
当Activity主题为Dialog时，WebView不显示。
猜测这个和WebView的onMeasure()实现过程有关。

##上传文件
###代码模板
重写openFileChooser()方法和onShowFileChooser()方法：

```java
private ValueCallback<Uri> mUploadMessage;
public ValueCallback<Uri[]> uploadMessage;
public static final int REQUEST_SELECT_FILE = 100;
private final static int FILECHOOSER_RESULTCODE = 1;

WebSettings mWebSettings = mWebView.getSettings();
    mWebSettings.setJavaScriptEnabled(true);
    mWebSettings.setSupportZoom(false);
    mWebSettings.setAllowFileAccess(true);
    mWebSettings.setAllowFileAccess(true);
    mWebSettings.setAllowContentAccess(true);

mWebView.setWebChromeClient(new WebChromeClient()
{
    // For 3.0+ Devices (Start)
    // onActivityResult attached before constructor
    protected void openFileChooser(ValueCallback uploadMsg, String acceptType)
    {
        mUploadMessage = uploadMsg;
        Intent i = new Intent(Intent.ACTION_GET_CONTENT);
        i.addCategory(Intent.CATEGORY_OPENABLE);
        i.setType("image/*");
        startActivityForResult(Intent.createChooser(i, "File Browser"), FILECHOOSER_RESULTCODE);
    }


    // For Lollipop 5.0+ Devices
    public boolean onShowFileChooser(WebView mWebView, ValueCallback<Uri[]> filePathCallback, WebChromeClient.FileChooserParams fileChooserParams)
    {
        if (uploadMessage != null) {
            uploadMessage.onReceiveValue(null);
            uploadMessage = null;
        }

        uploadMessage = filePathCallback;

        Intent intent = fileChooserParams.createIntent();
        try
        {
            startActivityForResult(intent, REQUEST_SELECT_FILE);
        } catch (ActivityNotFoundException e)
        {
            uploadMessage = null;
            Toast.makeText(getActivity().getApplicationContext(), "Cannot Open File Chooser", Toast.LENGTH_LONG).show();
            return false;
        }
        return true;
    }

    //For Android 4.1 only
    protected void openFileChooser(ValueCallback<Uri> uploadMsg, String acceptType, String capture)
    {
        mUploadMessage = uploadMsg;
        Intent intent = new Intent(Intent.ACTION_GET_CONTENT);
        intent.addCategory(Intent.CATEGORY_OPENABLE);
        intent.setType("image/*");
        startActivityForResult(Intent.createChooser(intent, "File Browser"), FILECHOOSER_RESULTCODE);
    }

    protected void openFileChooser(ValueCallback<Uri> uploadMsg)
    {
        mUploadMessage = uploadMsg;
        Intent i = new Intent(Intent.ACTION_GET_CONTENT);
        i.addCategory(Intent.CATEGORY_OPENABLE);
        i.setType("image/*");
        startActivityForResult(Intent.createChooser(i, "File Chooser"), FILECHOOSER_RESULTCODE);
    }
});
```

返回文件Uri：

```java
@Override
public void onActivityResult(int requestCode, int resultCode, Intent intent)
{
    if(Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP)
    {
        if (requestCode == REQUEST_SELECT_FILE)
        {
            if (uploadMessage == null)
                return;
            uploadMessage.onReceiveValue(WebChromeClient.FileChooserParams.parseResult(resultCode, intent));
            uploadMessage = null;
        }
    }
    else if (requestCode == FILECHOOSER_RESULTCODE)
    {
        if (null == mUploadMessage)
            return;
    // Use MainActivity.RESULT_OK if you're implementing WebView inside Fragment
    // Use RESULT_OK only if you're implementing WebView inside an Activity
        Uri result = intent == null || resultCode != MainActivity.RESULT_OK ? null : intent.getData();
        mUploadMessage.onReceiveValue(result);
        mUploadMessage = null;
    }
    else
        Toast.makeText(getActivity().getApplicationContext(), "Failed to Upload Image", Toast.LENGTH_LONG).show();
}
```

还可以参照Android内置浏览器的<a href="http://grepcode.com/file/repository.grepcode.com/java/ext/com.google.android/android-apps/4.0.4_r1.2/com/android/browser/UploadHandler.java#UploadHandler.openFileChooser%28android.webkit.ValueCallback%2Cjava.lang.String%29">UploadHandler.class</a>

openFileChooser()方法并不是WebChromeClient的对外开放的方法，因此这个方法会被混淆。openFileChooser()不被混淆方法：

```
-keepclassmembers class*extends android.webkit.WebChromeClient{
publicvoid openFileChooser(...);
}
```

使用openFileChooser()不管有没有选择文件都要调用onReceiveValue()，没有选择文件则onReceiveValue(null)。如果给返回值，那么，再次点击<input/>标签会报出empted to finish an input event but the input event receiver has already been disposed，无法再次点击弹出拍照或者图册选项框。

##Js弹窗无法弹出
设置WebChromeClient,onJsAlert()方法返回false：

```java
webView.setWebChromeClient(new WebChromeClient(){
    @Override
    public boolean onJsAlert(WebView view, String url, String message, JsResult result) {
        return false;
    }
});
```

##WebView调用浏览器打开链接
设置WebViewClient,shouldOverrideUrlLoading方法返回false：

```java
webView.setWebViewClient(new WebViewClient(){
    @Override
    public boolean shouldOverrideUrlLoading(WebView view, String url) {
        return false;
    }
});
```

##WebView不支持javasrcipt

设置支持javascript

```java
webView.getSettings().setJavaScriptEnabled(true);
```

##WebView设置Cookie
The CookieSyncManager is used to synchronize the browser cookie store between RAM and permanent storage. To get the best performance, browser cookies are saved in RAM. A separate thread saves the cookies between, driven by a timer.
To use the CookieSyncManager, the host application has to call the following when the application starts:

```java
CookieSyncManager.createInstance(context)
```

To set up for sync, the host application has to call

```java
CookieSyncManager.getInstance().startSync()
```

in Activity.onResume(), and call

```java
CookieSyncManager.getInstance().stopSync()
```

in Activity.onPause().
To get instant sync instead of waiting for the timer to trigger, the host can call

```java
CookieSyncManager.getInstance().sync()
```

The sync interval is 5 minutes, so you will want to force syncs manually anyway, for instance in onPageFinished(WebView, String). Note that even sync() happens asynchronously, so don't do it just as your activity is shutting down.

##WebView调用addJavascriptInterface方法的安全问题

在运行时获取文件列表：

```java
try {
    Process p=Runtime.getRuntime().exec(new String[]{"ls","/mnt/sdcard/"});
    InputStreamReader reader=new InputStreamReader(p.getInputStream());
    int c;

    while ((c=reader.read())!=-1){
        System.out.print((char)c);
    }
} catch (IOException e) {
    e.printStackTrace();
}
```

上面的代码获取Runtime实例，用命令获取sd卡目录。在web端可以通过反射获取Runtime实例，那么就给入侵提供了可能。

```html
//bridge为Android端传给web端的对象名称
bridge.getClass().forName("java.lang.Runtime").getMethod("getRuntime",null).invoke(null,null).exec(["ls","/mnt/sdcard/"]);
```

Android 4.2(API 17)以及之后版本，为了避免这个漏洞，web端只能调用加了@JavascriptInterfac注解的方法。

ps:<a href="http://blog.csdn.net/sbsujjbcy/article/details/50752595">Android JSBridge的原理与实现</a>




