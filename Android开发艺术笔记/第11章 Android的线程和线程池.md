# 第11章 Android的线程和线程池

除了Thread以外，在Android中可以扮演线程的角色还有很多，比如AsyncTask和IntentService，同时HandlerThread也是一种特殊的线程。

AsyncTask封装了线程池和Handler，它主要是为了方便在子线程中更新UI。HandlerThread是一种具有消息循环的线程，在它的内部可以使用Handler。IntentService是一个服务，IntentService内部采用HandlerThread来执行任务，当任务执行完毕后IntentService会自动退出。IntentService是一种服务，不容易被系统杀死从而可以尽量保证任务的执行，如果是一个后台线程，由于这个时候进程中没有活动的四大组件，那么这个进程的优先级就会降低，容易被系统杀死，这就是IntentService的优点。

## 11.1 主线程和子线程

从Android3.0开始系统要求网络访问必须在子线程执行，否则网络访问将会失败并抛出NetworkOnMainThreadException这个异常。

## 11.2 Android中的线程形态

### 11.2.1 AsyncTask

AsyncTask是一种轻量级的异步任务类，它可以在线程池中执行后台任务，然后把执行的进度和最终结果传递给主线程并在主线程中更新UI。

4个核心方法：

 1. onPreExecute()
 2. doInBackground(Params... params)在此方法中可以通过publishProgress方法来更新任务进度，publishProgress方法会调用onProgreeUpdate方法。
 3. onProgressUpdate(Progress...values)
 4. onPostEcecute(Result result)
 5. onCancelled(),任务被取消时被调用，onPostExecute()不会被调用

AsyncTask在使用中的一些限制条件：

 1. AsyncTask的类必须在主线程中加载
 2. AsyncTask的对象必须在主线程中创建
 3. execute方法必须在UI线程调用
 4. 不要直接调用onPreExecute()、onPostExecute、doInBackground、onProgressUpdate方法
 5. 一个AsyncTask对象只能执行一次，即只能调用一次execute方法
 6. Android1.6之前，AyncTask是串行执行任务的，Android1.6时AsyncTask开始采用线程池处理并行任务，但是从Android3.0开始，AsyncTask又采用一个线程来串行执行任务。在Android3.0以后的版本中，可以通过AsyncTask的executeOnExecutor方法来并行执行任务。

AsyncTask的工作泳道流程图：

![](https://www.github.com/wslaimin/blog/raw/master/pics/AsyncTask.png)

### 11.2.3 HandlerThread

HandlerThread继承了Thread，在run()方法中创建了Looper。

### 11.2.4 IntentService 

IntentService是一种特殊的Service，继承了Service。可以用来执行耗时任务(在非UI线程执行)，当任务执行后它会自动停止。

有两个停止服务的方法stopSelf()和stopSelf(int startId)。stopSelf()会立刻停止服务，stopSelf(int startId)会等待所有的的消息都处理完毕后才终止服务(原理是判断最近启动服务的id是否和startId相等)。

在onHandleIntent方法中处理耗时任务。HandlerThread继承Handler，所以一样是顺序处理消息，这意味着IntentService也是顺序执行后台任务。


 

 
   