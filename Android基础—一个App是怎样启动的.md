##                                      Android基础—一个App是怎样启动的

### 前言：

一个程序没有入口是不行的，在一个java程序中，程序的入口是main方法，但是在Android程序中，我们一般见到的是Activity的onCreat方法，没有见到main方法，那么一个App到底是怎么启动的，它的入口在哪？

Android应用程序的载体是APK文件，它本质上，是一个**资源和组件的容器**，APK文件和我们常见的**可执行文件**的区别在何处？

每个可执行文件运行在一个进程中，但是APK文件可能运行在一个单独的进程，也可以和其他APK运行在同一进程中，结合上面：

**Android系统的设计理念就是弱化进程，取而代之是组件的概念。**

 Android系统基于**Linux**系统之上 ， 而Linux系统的运行环境恰恰就是由**进程**组成。

 除了从Linux继承下来的东西外，Android还需要在**应用的Java层**建立一套框架来管理运行的组件。由于每个应用的配置都不相同，因此不能再Zygote中完全建立好再继承，只能在应用启动时创建。

 **这套框架就构成了Android应用的基础。** 

而我们今天的主角就是这套框架中的一个核心类， **ActivityThread** 。

先回答提出的问题，一个android程序是从**ActivityThread**启动的.

### **ActivityThread**

```java
 public final class ActivityThread {
 ...............................
 .............................
 
 public static void main(String[] args) {
        Trace.traceBegin(64L, "ActivityThreadMain");
        CloseGuard.setEnabled(false);
        Environment.initForCurrentUser();
        EventLogger.setReporter(new ActivityThread.EventLoggingReporter());
        File configDir = Environment.getUserConfigDirectory(UserHandle.myUserId());
        TrustedCertificateStore.setDefaultUserDirectory(configDir);
        Process.setArgV0("<pre-initialized>");
        Looper.prepareMainLooper();
        ActivityThread thread = new ActivityThread();
        thread.attach(false);
        if (sMainThreadHandler == null) {
            sMainThreadHandler = thread.getHandler();
        }
   Looper.loop();
.................................
.............................
}
```

是不是看到了一个熟悉的方法，Main方法。就是从这里启动的。

然后其他的不看，看到

```java
Looper.prepareMainLooper(); 
ActivityThread thread = new ActivityThread();
thread.attach(false);
Looper.loop();
```

这里开启了消息循环，并且创建了一个ActivityThread，这个就是我们的App的活动线程，然后又做了一个attach.

看看这个attach做了什么

```java
 private void attach(boolean system) {
        sCurrentActivityThread = this;
        this.mSystemThread = system;
        if (!system) {
            ViewRootImpl.addFirstDrawHandler(new Runnable() {
                public void run() {
                    ActivityThread.this.ensureJitEnabled();
                }
            });
            DdmHandleAppName.setAppName("<pre-initialized>", UserHandle.myUserId());
            RuntimeInit.setApplicationObject(this.mAppThread.asBinder());
               //注意这里
            final IActivityManager mgr = ActivityManager.getService();

            try {
                 //注意这里
                mgr.attachApplication(this.mAppThread);
            } catch (RemoteException var5) {
                throw var5.rethrowFromSystemServer();
            }

    
```

在注释的那个地方用ActivityManager的getService()获取到了一个IActivityManager

再点去看看

```java
 //ActivityManager.java
 
 public static IActivityManager getService() {
        return (IActivityManager)IActivityManagerSingleton.get();
    }
    
     private static final Singleton<IActivityManager> IActivityManagerSingleton = new Singleton<IActivityManager>() {
        protected IActivityManager create() {
            IBinder b = ServiceManager.getService("activity");
            IActivityManager am = Stub.asInterface(b);
            return am;
        }
    };
```

这里明显用了单例，注意

```java
 IBinder b = ServiceManager.getService("activity");
            IActivityManager am = Stub.asInterface(b);
```

**这里就是核心了，这里的ServiceManager并不是我们的App里面的，而是属于系统的，用系统的服务返回了一个Binder，Binder就是很典型的用来跨进程通信的。就是说我们调用系统的ServiceManager服务,并传入参数，返回一个Binder,然后用这个Binder来跨进程的创建一个我们自己的App的ActivityManager(活动管理)。**

然后继续看

```
 mgr.attachApplication(this.mAppThread);
```

attachApplication在这里的作用其实实际上是ActivityThread通过attach获取到，然后将applciationThread将其关联，把activity相关信息存储在applciationThread里面，apllicationThread的类为activity的各种状态做了相对应的准备工作

我的理解是进一步和APP自身的ApplicationThread关联起来了。

然后主要看看ApplicationThread里面到底有什么

```java
 private class ApplicationThread extends android.app.IApplicationThread.Stub {
        private static final String DB_INFO_FORMAT = "  %8s %8s %14s %14s  %s";
        private int mLastProcessState;

        private ApplicationThread() {
            this.mLastProcessState = -1;
        }

        private void updatePendingConfiguration(Configuration config) {
            synchronized(ActivityThread.this.mResourcesManager) {
                if (ActivityThread.this.mPendingConfiguration == null || ActivityThread.this.mPendingConfiguration.isOtherSeqNewer(config)) {
                    ActivityThread.this.mPendingConfiguration = config;
                }

            }
        }

        public final void schedulePauseActivity(IBinder token, boolean finished, boolean userLeaving, int configChanges, boolean dontReport) {
            int seq = ActivityThread.this.getLifecycleSeq();
            ActivityThread.this.sendMessage(finished ? 102 : 101, token, (userLeaving ? 1 : 0) | (dontReport ? 2 : 0), configChanges, seq);
        }

        public final void scheduleStopActivity(IBinder token, boolean showWindow, int configChanges) {
            int seq = ActivityThread.this.getLifecycleSeq();
            ActivityThread.this.sendMessage(showWindow ? 103 : 104, token, 0, configChanges, seq);
        }

        public final void scheduleWindowVisibility(IBinder token, boolean showWindow) {
            ActivityThread.this.sendMessage(showWindow ? 105 : 106, token);
        }

        public final void scheduleSleeping(IBinder token, boolean sleeping) {
            ActivityThread.this.sendMessage(137, token, sleeping ? 1 : 0);
        }

        public final void scheduleResumeActivity(IBinder token, int processState, boolean isForward, Bundle resumeArgs) {
            int seq = ActivityThread.this.getLifecycleSeq();
            this.updateProcessState(processState, false);
            ActivityThread.this.sendMessage(107, token, isForward ? 1 : 0, 0, seq);
        }

        public final void scheduleSendResult(IBinder token, List<ResultInfo> results) {
            ActivityThread.ResultData res = new ActivityThread.ResultData();
            res.token = token;
            res.results = results;
            ActivityThread.this.sendMessage(108, res);
        }

        public final void scheduleLaunchActivity(Intent intent, IBinder token, int ident, ActivityInfo info, Configuration curConfig, Configuration overrideConfig, CompatibilityInfo compatInfo, String referrer, IVoiceInteractor voiceInteractor, int procState, Bundle state, PersistableBundle persistentState, List<ResultInfo> pendingResults, List<ReferrerIntent> pendingNewIntents, boolean notResumed, boolean isForward, ProfilerInfo profilerInfo) {
            this.updateProcessState(procState, false);
            ActivityThread.ActivityClientRecord r = new ActivityThread.ActivityClientRecord();
            r.token = token;
            r.ident = ident;
            r.intent = intent;
            r.referrer = referrer;
            r.voiceInteractor = voiceInteractor;
            r.activityInfo = info;
            r.compatInfo = compatInfo;
            r.state = state;
            r.persistentState = persistentState;
            r.pendingResults = pendingResults;
            r.pendingIntents = pendingNewIntents;
            r.startsNotResumed = notResumed;
            r.isForward = isForward;
            r.profilerInfo = profilerInfo;
            r.overrideConfig = overrideConfig;
            this.updatePendingConfiguration(curConfig);
            ActivityThread.this.sendMessage(100, r);
        }

        public final void scheduleRelaunchActivity(IBinder token, List<ResultInfo> pendingResults, List<ReferrerIntent> pendingNewIntents, int configChanges, boolean notResumed, Configuration config, Configuration overrideConfig, boolean preserveWindow) {
            ActivityThread.this.requestRelaunchActivity(token, pendingResults, pendingNewIntents, configChanges, notResumed, config, overrideConfig, true, preserveWindow);
        }

```

我们可以看到一些很比较熟悉的方法名，比如scheduleLaunch，scheduleResume，scheduleStop

这难道不像我们的Activity的生命周期吗？而且每一个这样的方法最后都会sendMessage

```java
ActivityThread.this.sendMessage(100, r);
```

这不就是Handler机制吗

再来看看scheduleLaunchActivity这个方法。

```java
 public final void scheduleLaunchActivity(Intent intent, IBinder token, int ident, ActivityInfo info, Configuration curConfig, Configuration overrideConfig, CompatibilityInfo compatInfo, String referrer, IVoiceInteractor voiceInteractor, int procState, Bundle state, PersistableBundle persistentState, List<ResultInfo> pendingResults, List<ReferrerIntent> pendingNewIntents, boolean notResumed, boolean isForward, ProfilerInfo profilerInfo) {
            this.updateProcessState(procState, false);
            ActivityThread.ActivityClientRecord r = new ActivityThread.ActivityClientRecord();
            r.token = token;
            r.ident = ident;
            r.intent = intent;
            r.referrer = referrer;
            r.voiceInteractor = voiceInteractor;
            r.activityInfo = info;
            r.compatInfo = compatInfo;
            r.state = state;
            r.persistentState = persistentState;
            r.pendingResults = pendingResults;
            r.pendingIntents = pendingNewIntents;
            r.startsNotResumed = notResumed;
            r.isForward = isForward;
            r.profilerInfo = profilerInfo;
            r.overrideConfig = overrideConfig;
            this.updatePendingConfiguration(curConfig);
            ActivityThread.this.sendMessage(100, r);
        }
```

这里的 r 很引人注目，可以看到这里更新了r的配置，那么r 是什么呢？

```
 ActivityThread.ActivityClientRecord r = new ActivityThread.ActivityClientRecord();
```

```java
 static final class ActivityClientRecord {
        IBinder token;
        int ident;
        Intent intent;
        String referrer;
        IVoiceInteractor voiceInteractor;
        Bundle state;
        PersistableBundle persistentState;
        Activity activity;
        Window window;
        Activity parent = null;
        String embeddedID = null;
        NonConfigurationInstances lastNonConfigurationInstances;
        boolean paused = false;
        boolean stopped = false;
        boolean hideForNow = false;
 。。。。。。。。。。。。。。。。
 }
```

这里可以看到有了一个activity和一堆配置，我们所要启动的acticity就在这里面

回过头来，这个r 被sendMessage发了出去

那么就看接受的时候的方法

```java
  public void handleMessage(Message msg) {
            ActivityThread.ActivityClientRecord r;
            SomeArgs args;
            switch(msg.what) {
            case 100:
                Trace.traceBegin(64L, "activityStart");
                r = (ActivityThread.ActivityClientRecord)msg.obj;
                r.packageInfo = ActivityThread.this.getPackageInfoNoCheck(r.activityInfo.applicationInfo, r.compatInfo);
                ActivityThread.this.handleLaunchActivity(r, (Intent)null, "LAUNCH_ACTIVITY");
                Trace.traceEnd(64L);
                break;
            case 101:
                Trace.traceBegin(64L, "activityPause");
                args = (SomeArgs)msg.obj;
                ActivityThread.this.handlePauseActivity((IBinder)args.arg1, false, (args.argi1 & 1) != 0, args.argi2, (args.argi1 & 2) != 0, args.argi3);
                Trace.traceEnd(64L);
                break;
            case 102:
                Trace.traceBegin(64L, "activityPause");
                args = (SomeArgs)msg.obj;
                ActivityThread.this.handlePauseActivity((IBinder)args.arg1, true, (args.argi1 & 1) != 0, args.argi2, (args.argi1 & 2) != 0, args.argi3);
                Trace.traceEnd(64L);
                break;
            case 103:
                Trace.traceBegin(64L, "activityStop");
                args = (SomeArgs)msg.obj;
                ActivityThread.this.handleStopActivity((IBinder)args.arg1, true, args.argi2, args.argi3);
                Trace.traceEnd(64L);
                break;
            case 104:
                Trace.traceBegin(64L, "activityStop");
                args = (SomeArgs)msg.obj;
                ActivityThread.this.handleStopActivity((IBinder)args.arg1, false, args.argi2, args.argi3);
                Trace.traceEnd(64L);
                break;
            case 105:
                Trace.traceBegin(64L, "activityShowWindow");
                ActivityThread.this.handleWindowVisibility((IBinder)msg.obj, true);
                Trace.traceEnd(64L);
                break;
            case 106:
                Trace.traceBegin(64L, "activityHideWindow");
                ActivityThread.this.handleWindowVisibility((IBinder)msg.obj, false);
                Trace.traceEnd(64L);
                break;
            case 107:
                Trace.traceBegin(64L, "activityResume");
                args = (SomeArgs)msg.obj;
                ActivityThread.this.handleResumeActivity((IBinder)args.arg1, true, args.argi1 != 0, true, args.argi3, "RESUME_ACTIVITY");
                Trace.traceEnd(64L);
                break;
```

还记得最开始的时候 ，开启了Loop消息循环，不断收取消息，也就是说开始的一系列流程最终会到这里来。

这里就很清楚了，无非是根据message的参数来调用不同的方法，具体就是**通过发送时不同的状态，这边调用了不同的handlerXXXActivity方法** 

**那么其实到此为止，我门可以得出一个结论，**

**Application运行的过程当中，对于Activity的操作，状态转变，其实实际上是通过Handler消息机制来完成的，**

**Application当中只管去发， 由消息机制负责调用，因为在main方法当中我门的Looper轮训器是一直在进行轮训的**

然后我们在进一步看handleLaunchActivity里面干了什么

```java
private void handleLaunchActivity(ActivityThread.ActivityClientRecord r, Intent customIntent, String reason) {
        this.unscheduleGcIdler();
        this.mSomeActivitiesChanged = true;
        if (r.profilerInfo != null) {
            this.mProfiler.setProfiler(r.profilerInfo);
            this.mProfiler.startProfiling();
        }

        this.handleConfigurationChanged((Configuration)null, (CompatibilityInfo)null);
        if (!ThreadedRenderer.sRendererDisabled) {
            GraphicsEnvironment.earlyInitEGL();
        }

        WindowManagerGlobal.initialize();
        //注意这里
        Activity a = this.performLaunchActivity(r, customIntent);
      
    
    if (a != null) {
            r.createdConfig = new Configuration(this.mConfiguration);
            this.reportSizeConfigurations(r);
            Bundle oldState = r.state;
            this.handleResumeActivity(r.token, false, r.isForward, !r.activity.mFinished && !r.startsNotResumed, r.lastProcessedSeq, reason);
            if (!r.activity.mFinished && r.startsNotResumed) {
                this.performPauseActivityIfNeeded(r, reason);
                if (r.isPreHoneycomb()) {
                    r.state = oldState;
                }
```

```java
          //注意这里
        Activity a = this.performLaunchActivity(r, customIntent);
```

这里我们真正的发现了activity的影子，再进去看看performLaunchActivity干了什么

```java
   private Activity performLaunchActivity(ActivityThread.ActivityClientRecord r, Intent customIntent) {
  ..................................
                  activity.mCalled = false;
                if (r.isPersistable()) {
                    this.mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
                } else {
                    this.mInstrumentation.callActivityOnCreate(activity, r.state);
                }
                ...............
                
                }
```

在这里我们找到了OnCreat方法的影子了。

 也就是说，到目前为止我们能够明白，整个Application加载Activity的整套流程是怎么回事 

所以结果出来了，下面我来理一遍（我自己的理解）：

**手机点击一个App图标，然后就进入ActivityThread的Main方法，然后通过attach把App的进程与系统进程关联（这里用到**了Binder跨进程），然后**系统就找到ApplicationThread** **,开始调用里面的各种schedule方法，app是哪个状态就调用对**应**的哪个方法，然后在schedule方法里面创建了一个Acivity的模型（ActivityClientRecord），然后配置参数，然后用handler的sendmessage发出去，然后在当前app里接受消息，根据不同状态回调相应的方法，这些回调的方法其实就是对应Activity的生命周期。**































