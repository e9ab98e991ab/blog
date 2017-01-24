#Working with System Permissions笔记

#Declaring Permissions
在Manifest里声明的权限，不涉及用户隐私的权限，系统自动授权，设计隐私的权限，系统会询问用户，让用户授权。

Android5.1和更低版本，用户在安装时给app授权，包括涉及用户隐私的权限；Android6.0和更高版本涉及用户隐私的权限在运行时授权。

App只需要直接执行相关动作的权限。比如，app需要获取联络人电话，需要READ_CONTACTS权限，但是通过intent请求Contact app获取联络人电话，不需要任何权限，但是，Contact app需要这个权限。

#Requesting Permissions at Run Time
Android6.0开始，用户可以在Settings里取消应用权限。

系统权限分为两类，normal和dangerous：

 - normal权限不直接对用户隐私造成危险，在manifest文件中声明，系统自动授权。
 - dangerous权限能够接触到用户机密数据，在manifest文件声明，用户还必须显示授权。

系统版本和target SDK对权限声明的影响：

 - 如果设备运行的是Android 5.1或更低版本，或者target SDK是22或更低：dangerous权限在app安装过程授权，不授权不能安装app。
 - 如果设备运行Android 6.0或更高版本，并且target SDK是23或者更高:dangerous权限在运行时授权。

>注意：从Android 6.0开始(API level 23)，用户可以随时撤销权限，即使app target SDK低于23.
 
检查权限：

```java
// Assume thisActivity is the current activity
int permissionCheck = ContextCompat.checkSelfPermission(thisActivity,
        Manifest.permission.WRITE_CALENDAR);
```

用户之前拒绝授权，之后又使用需要该权限的功能，可以通过shouldShowRequestPermissionRationable()方法判断是否要对权限作出解释。如果之前请求该权限被拒绝，这个方法返回true。

>注意：如果用户拒绝授权并且选择Don't ask again，这个方法返回false。设备禁止app使用某个权限也返回false。

请求权限：

```java
// Here, thisActivity is the current activity
if (ContextCompat.checkSelfPermission(thisActivity,
                Manifest.permission.READ_CONTACTS)
        != PackageManager.PERMISSION_GRANTED) {

    // Should we show an explanation?
    if (ActivityCompat.shouldShowRequestPermissionRationale(thisActivity,
            Manifest.permission.READ_CONTACTS)) {

        // Show an explanation to the user *asynchronously* -- don't block
        // this thread waiting for the user's response! After the user
        // sees the explanation, try again to request the permission.

    } else {

        // No explanation needed, we can request the permission.

        ActivityCompat.requestPermissions(thisActivity,
                new String[]{Manifest.permission.READ_CONTACTS},
                MY_PERMISSIONS_REQUEST_READ_CONTACTS);

        // MY_PERMISSIONS_REQUEST_READ_CONTACTS is an
        // app-defined int constant. The callback method gets the
        // result of the request.
    }
}
```

处理权限请求结果

```java
@Override
public void onRequestPermissionsResult(int requestCode,
        String permissions[], int[] grantResults) {
    switch (requestCode) {
        case MY_PERMISSIONS_REQUEST_READ_CONTACTS: {
            // If request is cancelled, the result arrays are empty.
            if (grantResults.length > 0
                && grantResults[0] == PackageManager.PERMISSION_GRANTED) {

                // permission was granted, yay! Do the
                // contacts-related task you need to do.

            } else {

                // permission denied, boo! Disable the
                // functionality that depends on this permission.
            }
            return;
        }

        // other 'case' lines to check for other
        // permissions this app might request
    }
}
```

permission group：权限组，如果请求权限组中的某个权限，用户同意授权，那么之后再请求同一权限组的其他权限，系统自动授权。比如用户同意授权READ_CONTACTS，之后请求WRITE_CONTACTS权限，系统自动授权。最好，对每个权限都显示请求，就算是同一组的权限，而且之后的发布版本权限组可能会发生改变。

#Permissions Usage Notes
按组列出权限:

```
$ adb shell pm list permission -d -g
```

选项-d表示dangerous权限，-g表示按组列出。

同意授权或撤销授权:

```
$ adb shell pm [grant|revoke] <permission-name> ...
```

