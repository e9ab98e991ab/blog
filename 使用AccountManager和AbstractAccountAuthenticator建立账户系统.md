#使用AccountManager和AbstractAccountAuthenticator建立账户系统
[TOC]
##为什么要使用AccountManager和AccountAuthenticatr
建立账户管理系统，可以使用SharedPreference来存储、更新、删除AuthToken，也可以用来存储、更新、删除账户和密码。既然这样，为什么还要使用AccountManager和AccountAuthenticator?
原因有一下几点：
1.AccountManager是Android系统提供的账户管理框架，十分强大和方便，可以管理不同类型账户，不同类型AuthenToken
2.AccountAuthenticator把账户的验证过程、AuthToken的获取过程分离出来，降低程序的耦合性
3.使用AccountAuthenticator会在"设置"中添加一个账户入口，感觉很酷炫。
![](https://github.com/wslaimin/blog/account.png)

##AccountManager和Authenticator之间的关系
AccountManager和Authenticator的方法相对应，比如，AccountManager的addAccount()方法会调用Authenticator的addAccount()方法，Authenticator的方法会返回一个Bundle给AccountManager处理。具体细节后面会介绍。
##使用AccountManager管理账号
顾名思义，AccountManager是用来管理用户账户。使用AccountManager有两个要点：Bundle key常量的含义、如何使用账户管理方法。
AccountManager定义了许多Bundle Key常量，可以用作Intent的key用于Activity之间传递参数；也可以作为Authenticator返回Bundle的Key，这种情况，AccountManager会根据Key的情况作出相应操作，比如，跳转到验证身份页面，返回Accont Type,AuthToken,Account Name(getAuthenToken()有返回AuthToken时必须返回Account Type和Account Name，不然会提示“the type and name should not be empty”)等。

介绍一些常用Bundle key:
|Key                                                |含义|
|---                                                |----|
|AccountManager.KEY_ACCOUNT_AUTHENTICATOR_RESPONSE  |可以调用onResult()和onError()来相应用户提供的信息|          
|AccountManager.KEY_INTENT                          |开启新Activity和用户进行交互|       
|AccountManager.KEY_AUTHTOKEN                       |令牌|
|AccountManager.KEY_ACCOUNT_TYPE                    |账户类型|
|AccountManager.KEY_ACCOUNT_NAME                    |账户名|
|AccountManager.KEY_ERROR_CODE                      |错误码(必须大于0)|
|AccountManager.KEY_ERROR_MESSAGE                   |错误信息(必须和错误码一起使用)|

介绍一些常用AccountManager管理账户方法：
获取AuthToken：
```java
@param account 账户
@param authTokenType token类型
@param options 额外的数据
@param activity 用来开启另外的Activity
@param callback 结果回调
@param 回调的线程，null时为主线程
public AccountManagerFuture<Bundle> getAuthToken(
            final Account account, final String authTokenType, final Bundle options,
            final Activity activity, AccountManagerCallback<Bundle> callback, Handler handler) 
```
获取账户列表：
```java
@param type 账户列表类型
public Account[] getAccountsByType(String type)
```
从AccountManager的缓存中移除AuthToken:
```java
@param accountType 账户类型
@param authToken 令牌
public void invalidateAuthToken(final String accountType, final String authToken) 
```
获取密码：
```java 
@param account 账户
public String getPassword(final Account account) 
```
接下来是重点，创建自己的Authenticator，用来验证用户信息或者与从服务器获取信息。
##创建Authenticator
步骤：
1.继承AbstractAcccountAuthenticator
2.重写addAccount(),getAuthenToken()方法。如果有需要还可以重写其他方法
重写addAccount()方法：
```java
 @Override
    public Bundle addAccount(AccountAuthenticatorResponse response, String accountType, String authTokenType, String[] requiredFeatures, Bundle options) throws NetworkErrorException {
        Intent intent=new Intent(mContext,AuthenticatorActivity.class);
        intent.putExtra(AccountManager.KEY_ACCOUNT_AUTHENTICATOR_RESPONSE,response);
        Bundle bundle=new Bundle();
        bundle.putParcelable(AccountManager.KEY_INTENT,intent);
        return bundle;
    }
```
重写getAuthenToken()方法，要注意，如果之前成功获取过AuthenToken会缓存，之后不会在调用getAuthenToken()方法，除非调用invalidateAuthenToken()：
```java
@Override
    public Bundle getAuthToken(AccountAuthenticatorResponse response, Account account, String authTokenType, Bundle options) throws NetworkErrorException {
        //可以请求服务器获取token,这里为了简单直接返回
        Bundle bundle;
        if(!authTokenType.equals(Constants.AUTH_TOKEN_TYPE)){
            bundle=new Bundle();
            //没有error_code的情况,不会抛出异常
            //bundle.putInt(AccountManager.KEY_ERROR_CODE,1);
            bundle.putString(AccountManager.KEY_ERROR_MESSAGE,"invalid authToken");
            return bundle;
        }

        AccountManager am=AccountManager.get(mContext);
        String psw=am.getPassword(account);
        if(!TextUtils.isEmpty(psw)){
            //链接服务器获取token
            Random random=new Random();
            bundle=new Bundle();
            bundle.putString(AccountManager.KEY_AUTHTOKEN,random.nextLong()+"");
            //不返回name和type会报错“the type and name should not be empty”
            bundle.putString(AccountManager.KEY_ACCOUNT_TYPE,account.type);
            bundle.putString(AccountManager.KEY_ACCOUNT_NAME,account.name);
            return bundle;
        }


        bundle=new Bundle();
        Intent intent=new Intent(mContext,AuthenticatorActivity.class);
        bundle.putParcelable(AccountManager.KEY_INTENT,intent);
        intent.putExtra(AccountManager.KEY_ACCOUNT_TYPE,account.type);
        intent.putExtra(AccountManager.KEY_ACCOUNT_NAME,account.name);
        intent.putExtra(AccountManager.KEY_ACCOUNT_AUTHENTICATOR_RESPONSE,response);
        return bundle;
    }
```
3.为Authenticator创建Service
```java
public class AuthenticatorService extends Service{
    private MyAuthenticator mAuthenticator=new MyAuthenticator(this);

    @Nullable
    @Override
    public IBinder onBind(Intent intent) {
        return mAuthenticator.getIBinder();
    }
}
```
4.为了把Authenticator组件添加到账户管理框架，需要添加Metadata文件描述组件，在res/xml/目录下声明组件
```xml
<account-authenticator xmlns:android="http://schemas.android.com/apk/res/android"
    android:accountType="com.lm.example.accountmanagersample"
    android:icon="@drawable/ic_delete"
    android:smallIcon="@drawable/ic_delete"
    android:label="@string/app_name"
/>
```
>accountType很重要，用来唯一标识Authenticator，AccountManager的方法中有accountType的参数需要和此处保持一致。

5.在Manifest文件声明之前创建的Service
```xml
<service android:name=".AuthenticatorService">
            <intent-filter>
                <action
                    android:name="android.accounts.AccountAuthenticator" />
            </intent-filter>
            <meta-data
                android:name="android.accounts.AccountAuthenticator"
                android:resource="@xml/authenticator"/>
        </service>
```
##源码分析
从源码角度分析AccountManager和Authenticator之间的关系，主要介绍AccountManager的addAccount()和getAuthToken()方法。
首先分析addAccount():


