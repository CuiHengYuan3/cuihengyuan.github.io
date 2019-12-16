#                                                      Android-进程保活

## 进程优先级

官方文档： https://developer.android.google.cn/guide/components/activities/process-lifecycle?hl=zh-cn 

​     **前台进程** 

如果满足以下任一条件，则认为该进程处于前台：

- 它正在`Activity` 与用户进行交互的屏幕顶部运行（其 `onResume()`方法已被调用）。
- 它具有`BroadcastReceiver`当前正在运行的（其`BroadcastReceiver.onReceive()`方法正在执行）。
- 它有一个`Service`当前在其回调之一执行代码（`Service.onCreate()`， `Service.onStart()`，或 `Service.onDestroy()`）。

1. 系统中将永远只有少数这样的进程，并且只有在内存太低以至于这些进程都无法继续运行的情况下，这些进程才最终被杀死。 

2. **可见进程** 

不在前台，但是仍然会影响用户可见的内容。 将其删除将对用户体验产生明显的负面影响。在以下情况下，认为过程可见： 

- 它正在运行一个`Activity` 在屏幕上对用户可见但在前台不可见的（`onPause()`已调用其 方法）。例如，如果前景活动显示为允许在其后面看到先前活动的对话框，则可能会发生这种情况。
- 它具有一个`Service`作为前台服务运行的服务，`Service.startForeground()`该服务通过（要求系统将服务视为用户已意识到或对用户基本可见的东西）来运行。
- 它托管了系统正在用于用户了解的特定功能的服务，例如动态壁纸，输入法服务等。

与前台进程相比，系统中运行的这些进程的数量不受限制，但仍受到相对控制。这些进程被认为非常重要，除非必须这样做才能使所有前台进程保持运行，否则它们不会被杀死。

​      **服务进程**

托管已经调用了startService() 的Service 。 尽管这些进程对用户不是直接可见的，但是它们通常是在做用户关心的事情（**例如后台网络数据上传或下载**），因此，除非没有足够的内存来保留所有进程，否则系统将始终保持这些进程运行前台和可见进程。 

长时间运行（例如30分钟或更长时间）的服务可能会被降级，以使其进程下降到下面描述的缓存LRU列表。这有助于避免由于内存泄漏或其他问题而导致运行时间非常长的服务占用大量RAM的情况，从而使系统无法有效利用缓存的进程。

​     **后台进程**

这些进程通常包含一个或多个`Activity`当前对用户不可见的实例（该 `onStop()`方法已被调用并返回）。只要他们正确实现了Activity生命周期，则当系统终止此类进程时，它不会影响用户返回该应用程序时的体验：当在应用程序中重新创建关联的活动时，它可以恢复以前保存的状态。新过程。

这些进程保存在伪LRU列表中，列表中的最后一个进程是第一个杀死该进程以回收内存。在此列表上进行排序的确切策略是平台的实现细节，但通常它将尝试在其他类型的进程之前保留更多有用的进程（一个托管用户的家庭应用程序，他们看到的最后一个活动等）。还可以应用其他用于终止进程的策略：对允许的进程数量的硬性限制，对进程可以保持连续缓存的时间量的限制等。

## LowMemoryKiller

引入：

进程的启动分**冷启动**和**热启动**，当用户退出某一个进程的时候，并不会真正的将进程退出，而是将这个进程放到**后台**，以便下次启动的时候可以马上启动起来，这个过程名为热启动，这也是Android的设计理念之一。这个机制会带来一个问题，每个进程都有自己独立的内存地址空间，随着应用打开数量的增多,系统已使用的内存越来越大，就很有可能导致系统内存不足。为了解决这个问题，系统引入LowmemoryKiller(简称lmk)管理所有进程，根据一定策略来kill某个进程并释放占用的内存，保证系统的正常运行

可以演示。

原理：

所有应用进程都是从linux孵化出来的，记录在AMS中mLruProcesses列表中，由AMS进行统一管理，AMS中会根据进程的状态更新进程对应的oom_adj值，这个值会通过文件传递到kernel中去，kernel有个低内存回收机制，在内存达到一定阀值时会触发清理oom_adj值高的进程腾出更多的内存空间

 ![img](https://img-blog.csdn.net/20180715101905399?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3d4ajI4MDMwNjQ1MQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70) 

可见oom_adj的值越小，优先级就越高，就越难被杀死。

**所以进程保活的最重要的就是想办法把oom_adj值减小，让进程优先级变高**



## 常见的（流氓）保活方法



## Activity提升进程权限

监听锁屏事件，当锁屏的时候就启动一个像素为1的Activiy ,然后在解锁的时候销毁。因为在启动这个activity的时候这个新activity就到前台了，虽然用户看不见。提升了进程优先级。

代码

```java
//这只是那个像素为1的activity的代码，还需要在mainActivity中注册广播，监听广播

public class KeepActivity extends Activity {
   
    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        Log.e(TAG,"启动Keep");
        Window window = getWindow();
        //设置这个activity在左上角
        window.setGravity(Gravity.START | Gravity.TOP);
        WindowManager.LayoutParams attributes = window.getAttributes();
        //宽高为1
        attributes.width = 1;
        attributes.height = 1;
        //起始位置左上角
        attributes.x = 0;
        attributes.y = 0;
        window.setAttributes(attributes);
 
    }


}

```



## Service提升进程权限

这种方法的核心就是启动一个前台Service,然后让这个服务不可见。

 当应用程序处于前台时，它可以自由创建和运行前台和后台服务。当应用进入后台时，它会有几分钟的窗口，仍允许其创建和使用服务。在该窗口结束时，该应用程序被视为*处于空闲状态*。此时，系统停止应用程序的后台服务，就像该应用程序已调用服务的`Service.stopSelf()`方法一样。 

 在Android 8.0之前，创建前台服务的通常方法是创建后台服务，然后将该服务提升为前台。使用Android 8.0会很复杂。系统不允许后台应用创建后台服务。因此，Android 8.0引入了新方法`startForegroundService()`以在前台启动新服务。系统创建服务后，应用程序将有五秒钟的时间调用服务的`startForeground()`方法以显示新服务的用户可见通知。如果该应用程序*未*`startForeground()`在该期限内调用，则系统将停止该服务并声明该应用程序为 [ANR](https://developer.android.google.cn/training/articles/perf-anr.html)。 

```java
public class ForegroundService extends Service {
    @Nullable
    @Override
    public IBinder onBind(Intent intent) {
        return null;
    }

    @Override
    public void onCreate() {
        super.onCreate();
           //当API大于26
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
            NotificationChannel channel = new NotificationChannel("deamon", "deamon",
                    NotificationManager.IMPORTANCE_LOW);
            NotificationManager manager = (NotificationManager) getSystemService(Context.NOTIFICATION_SERVICE);
            if (manager == null)
                return;
            manager.createNotificationChannel(channel);

            Notification notification = new NotificationCompat.Builder(this, "deamon").setAutoCancel(true).setCategory(
                    Notification.CATEGORY_SERVICE).setOngoing(true).setPriority(
                    NotificationManager.IMPORTANCE_LOW).build();
            startForeground(10, notification);
        } else if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.JELLY_BEAN_MR2) {
            //如果 大于18，小于26 的设备 启动一个Service startForeground给相同的id
            //然后结束那个Service，这样第一个服务会运行，而且通知图标会消失
            startForeground(10, new Notification());
            startService(new Intent(this, InnnerService.class));
        } else {
            //小于18，直接就一个默认的Notification,通知图标不会消失
            startForeground(10, new Notification());
        }
    }

    public static class InnnerService extends Service {

        @Override
        public void onCreate() {
            super.onCreate();
            startForeground(10, new Notification());
            stopSelf();
        }

        @Nullable
        @Override
        public IBinder onBind(Intent intent) {
            return null;
        }
    }
}

```

## 广播拉活

在发生特定系统事件时，系统会发出广播，通过在 AndroidManifest 中**静态注册**对应的广播监听器，即可在发生响应事件时拉活。但是从android 7.0开始，对广播进行了限制，而且在8.0更加严格

从Android 8.0（API级别26）开始，系统对清单声明的接收者施加了其他限制。

如果您的应用程序针对Android 8.0或更高版本，则您不能使用清单为大多数隐式广播（不专门针对您的应用程序的广播）声明接收方。也就无法唤醒了。

可以被静态注册的列表（都被筛选了，这些留下来的基本都不适合用来进程保活）

 https://developer.android.google.cn/guide/components/broadcast-exceptions.html 

## 全家桶保活

有多个app在用户设备上安装，只要开启其中一个就可以将其他的app也拉活。比如手机里装了手Q、QQ空间、兴趣部落等等，那么打开任意一个app后，其他的app也都会被唤醒。

## Service机制(Sticky)拉活

将 Service 设置为 START_STICKY，利用系统机制在 Service 死掉后自动拉活

其实默认就是Sticky的。没什么用

## 





