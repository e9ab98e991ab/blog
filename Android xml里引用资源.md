# Android xml里引用资源

##语法

```xml
@[package:]type/name
```

@表示引用资源,声明这是一个资源引用,name是资源名。 

##引用id

一般我们应该用"@+id/"来定义一个id，然后用@id来引用一个id，但是现在我发现apps/settings/res/layout/preferenc_progress.xml中有个"@+android:id/title"，怎么理解它？怎么用？ 

加上android：表示引用android.R.id里面定义的id资源，如果android.R.id里面确实有title这个id资源，就直接使用它，如果没有的话就在当前应用的R.id中产生一个title标识。

##引用字符串

@android:string表明引用的系统的(android.*)资源 
@string表示引用应用内部资源 

##引用属性

?表示引用主题属性

使用主题属性 : 

```xml
<?xml version="1.0" encoding="utf-8"?> 
<TextView id="text" 
    xmlns:android="http://schemas.android.com/apk/res/android" 
    android:layout_width="fill_parent"
    android:layout_height="fill_parent" 
    android:textColor="?android:attr/actionMenuTextColor" 
    android:text="@string/hello_world" /> 
```

>android框架内的themes.xml文件中`<style name="Theme">`以及子theme预先定义了许多属性值。这些属性值可以通过?android:attr/name的方式来引用。




