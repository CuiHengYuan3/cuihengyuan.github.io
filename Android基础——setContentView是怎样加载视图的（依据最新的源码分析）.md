##      Android基础——setContentView是怎样加载视图的（依据最新的源码分析）

​     我们一步步来看

首先我们点击onCreat中的setContentView方法。

```java
//AppCompatActivity.java
    @Override
    public void setContentView(@LayoutRes int layoutResID) {
        getDelegate().setContentView(layoutResID);
    }

```

发现调用了AppCompatActivity的getDelegata()

来看看这个方法返回了什么

```java
//AppCompatActivity.java
@NonNull
    public AppCompatDelegate getDelegate() {
        if (mDelegate == null) {
            mDelegate = AppCompatDelegate.create(this, this);
        }
        return mDelegate;
    }

```

返回了一个mDelegate,这个mDelegate是由AppCompatDelegate的一个create方法创造出来的对象。

看看这个方法

```java
//AppCompatDelegate.java
@NonNull
    public static AppCompatDelegate create(@NonNull Activity activity,
            @Nullable AppCompatCallback callback) {
        return new AppCompatDelegateImpl(activity, callback);
    }
```

可以看到返回了一个AppCompatDelegateImpl对象。

```java
class AppCompatDelegateImpl extends AppCompatDelegate
        implements MenuBuilder.Callback, LayoutInflater.Factory2 {
    
    ........................
}
```

而AppCompatDelegateImpl 又是继承了 AppCompatDelegate，所以根据多态

其实是调用的AppCompatDelegateImpl对象的setContentView方法

所以我们来看AppCompatDelegateImpl对象的setContentView方法

```java
//AppCompatDelegateImpl.java
@Override
    public void setContentView(int resId) {
        ensureSubDecor();
        ViewGroup contentParent = mSubDecor.findViewById(android.R.id.content);
        contentParent.removeAllViews();
        LayoutInflater.from(mContext).inflate(resId, contentParent);
        mAppCompatWindowCallback.getWrapped().onContentChanged();
    }

```

这里就来了，我们先不看第一个ensureSubDecor()方法，看后面几行，首先在mSubDecor中找到了一个id为content的ViewGroup,然后把这个ViewGroup中的所有View移除，然后加载布局文件到这个ViewGroup中。

这些都还可以看懂，逻辑也没什么问题，可关键是这个mSubDecor是怎么来的？

我们来看看

```java
private ViewGroup mSubDecor;
```

mSubDecor是AppCompatDelegateImpl的一个私有成员变量

那在setContentView方法用它之前，什么时候初始化的呢？答案很明确了，就是在我们没有看的setContentView方法的第一个方法ensureSubDecor()中的。

我们点进去看看

```java
//AppCompatDelegateImpl.java
private void ensureSubDecor() {
        if (!mSubDecorInstalled) {
            mSubDecor = createSubDecor();

            // If a title was set before we installed the decor, propagate it now
            CharSequence title = getTitle();
            if (!TextUtils.isEmpty(title)) {
                if (mDecorContentParent != null) {
                    mDecorContentParent.setWindowTitle(title);
                } else if (peekSupportActionBar() != null) {
                    peekSupportActionBar().setWindowTitle(title);
                } else if (mTitleView != null) {
                    mTitleView.setText(title);
                }
            }

            applyFixedSizeWindow();

            onSubDecorInstalled(mSubDecor);

            mSubDecorInstalled = true;

            // Invalidate if the panel menu hasn't been created before this.
            // Panel menu invalidation is deferred avoiding application onCreateOptionsMenu
            // being called in the middle of onCreate or similar.
            // A pending invalidation will typically be resolved before the posted message
            // would run normally in order to satisfy instance state restoration.
            PanelFeatureState st = getPanelState(FEATURE_OPTIONS_PANEL, false);
            if (!mIsDestroyed && (st == null || st.menu == null)) {
                invalidatePanelMenu(FEATURE_SUPPORT_ACTION_BAR);
            }
        }
    }

```

其他的都没啥可看的，只有这个才是关键

```java
  mSubDecor = createSubDecor();
```

mSubDecor就是在这里初始化的，我们点进去看看

```java
 private ViewGroup createSubDecor() {
     //这里很熟徐，把 AppCompatTheme的属性取出来，然后requestWindowFeature
     TypedArray a = mContext.obtainStyledAttributes(R.styleable.AppCompatTheme);
       if (a.getBoolean(R.styleable.AppCompatTheme_windowNoTitle, false)) {
            requestWindowFeature(Window.FEATURE_NO_TITLE);
        } else if (a.getBoolean(R.styleable.AppCompatTheme_windowActionBar, false)) {
            // Don't allow an action bar if there is no title.
            requestWindowFeature(FEATURE_SUPPORT_ACTION_BAR);
        }
        if (a.getBoolean(R.styleable.AppCompatTheme_windowActionBarOverlay, false)) {
            requestWindowFeature(FEATURE_SUPPORT_ACTION_BAR_OVERLAY);
        }
        if (a.getBoolean(R.styleable.AppCompatTheme_windowActionModeOverlay, false)) {
            requestWindowFeature(FEATURE_ACTION_MODE_OVERLAY);
        }
        mIsFloating = a.getBoolean(R.styleable.AppCompatTheme_android_windowIsFloating, false);
        a.recycle();

       // Now let's make sure that the Window has installed its decor by retrieving it
        ensureWindow();
        mWindow.getDecorView();

        final LayoutInflater inflater = LayoutInflater.from(mContext);
        ViewGroup subDecor = null;

       if (!mWindowNoTitle) {
            if (mIsFloating) {
                // If we're floating, inflate the dialog title decor
                subDecor = (ViewGroup) inflater.inflate(
                        R.layout.abc_dialog_title_material, null);

                // Floating windows can never have an action bar, reset the flags
                mHasActionBar = mOverlayActionBar = false;
            } else if (mHasActionBar) {
                /**
                 * This needs some explanation. As we can not use the android:theme attribute
                 * pre-L, we emulate it by manually creating a LayoutInflater using a
                 * ContextThemeWrapper pointing to actionBarTheme.
                 */
                TypedValue outValue = new TypedValue();
                mContext.getTheme().resolveAttribute(R.attr.actionBarTheme, outValue, true);

                Context themedContext;
                if (outValue.resourceId != 0) {
                    themedContext = new ContextThemeWrapper(mContext, outValue.resourceId);
                } else {
                    themedContext = mContext;
                }

                if (mDecorContentParent == null) {
            mTitleView = (TextView) subDecor.findViewById(R.id.title);
        }

                //..................一堆判断，让subDecor初始化为不同的视图
    
          final ContentFrameLayout contentView = (ContentFrameLayout) subDecor.findViewById(
                R.id.action_bar_activity_content);

                  // Change our content FrameLayout to use the android.R.id.content id.
            // Useful for fragments.
            windowContentView.setId(View.NO_ID);
            contentView.setId(android.R.id.content);
                 // Now set the Window's content view with the decor
        mWindow.setContentView(subDecor);
                      return subDecor;
    }

```

首先把 AppCompatTheme的属性取出来，然后requestWindowFeature，这就是为什么设置状态栏要在setContentView之前写，因为状态栏的设置就是在setContentView中进行的。

然后向下走看

```java
  final LayoutInflater inflater = LayoutInflater.from(mContext);
    ViewGroup subDecor = null;
```
这里用Context初始化了一个inflater(布局加载器)

然后生命了一个ViewGroup：subDecor，这个就是这个方法最终返回的ViewGroup,也就是它就是mSubDecor

```java
     final ContentFrameLayout contentView = (ContentFrameLayout) subDecor.findViewById(
                R.id.action_bar_activity_content);

                  // Change our content FrameLayout to use the android.R.id.content id.
            // Useful for fragments.
            windowContentView.setId(View.NO_ID);
            contentView.setId(android.R.id.content);
                 // Now set the Window's content view with the decor
        mWindow.setContentView(subDecor);
```
然后最重要的来了，这个contentView在已经被初始化的subDecor中被找到，contentView是一个ContentFrameLayout，ContentFrameLayout继承了FrameLayout，然后被设置id为content ,所以我们到此知道了，我们的布局都是在这里面的。

然后mWindow.setContentView(subDecor) ，最后看看它干了些什么.

首先我们看到setContentView是Window类的一个抽象方法，那么我们就去找实现。



```java
 <p>The only existing implementation of this abstract class is
 * android.view.PhoneWindow, which you should instantiate when needing a
 * Window.
 */
public abstract class Window {
public abstract void setContentView(View view);
    。。。。。。。。。。。。。。。。。。。。
}
```

通过注释我们发现，Window类只有一个唯一的实现PhoneWindow。

所以我们知道这里肯定是调用的PhoneWindow方法，并且传入了subDecor，并且里面有一个id为content的fragmentlayout.还有一个id为title的titleView.(通过各种判断可知，这个view可能通过配置而移除)



```java
//PhoneWindow.java
public void setContentView(View view) {
        this.setContentView(view, new LayoutParams(-1, -1));
    }

    public void setContentView(View view, LayoutParams params) {
        if (this.mContentParent == null) {
            this.installDecor();
        } else if (!this.hasFeature(12)) {
            this.mContentParent.removeAllViews();
        }

        if (this.hasFeature(12)) {
            view.setLayoutParams(params);
            Scene newScene = new Scene(this.mContentParent, view);
            this.transitionTo(newScene);
        } else {
            this.mContentParent.addView(view, params);
        }

        this.mContentParent.requestApplyInsets();
        android.view.Window.Callback cb = this.getCallback();
        if (cb != null && !this.isDestroyed()) {
            cb.onContentChanged();
        }

        this.mContentParentExplicitlySet = true;
    }

```



调用了installDecor();

```java
 private void installDecor() {
        this.mForceDecorInstall = false;
        if (this.mDecor == null) {
            this.mDecor = this.generateDecor(-1);
            this.mDecor.setDescendantFocusability(262144);
            this.mDecor.setIsRootNamespace(true);
            if (!this.mInvalidatePanelMenuPosted && this.mInvalidatePanelMenuFeatures != 0) {
                this.mDecor.postOnAnimation(this.mInvalidatePanelMenuRunnable);
            }
        } else {
            this.mDecor.setWindow(this);
        }

        if (this.mContentParent == null) {
            this.mContentParent = this.generateLayout(this.mDecor);
            this.mDecor.makeOptionalFitsSystemWindows();
        }
     
```

这里很明显是初始化两个东西，一个为mDecor，一个为mContentParent

**我们看到了mDecor是一个DecorView**

**mContentParent是一个ViewGroup**

透过注释的翻译，其实我们就能很明确知道这两个是用来干嘛的

// This is the view in which the window contents are placed. It is either（这是窗口内容放置的视图）

// mDecor itself, or a child of mDecor where the contents go.(它要么是mDecor本身，要么是mDecor的子类的内容。)

//This is the top-level view of the window, containing the window decor.(**这是在窗口当中的顶层View，包含窗口的decor**)

 一个代表的是顶层view，一个用来装他下面的视图内容 

 ![img](https://upload-images.jianshu.io/upload_images/12625345-498252c0a5506b4d.png?imageMogr2/auto-orient/strip|imageView2/2/format/webp) 

所以到此为此我们就彻底把布局弄清楚了，installDecor方法根据配置加载系统的布局容器和根布局，我们的布局是在一个id为content的framelayout中加载的。

好吧，其实说了半天其实就是想说，我们的布局是加载到系统为我们创建的布局之上的。

然后我们回到开始的地方。

```java
//AppCompatDelegateImpl.java
@Override
    public void setContentView(int resId) {
        ensureSubDecor();
        ViewGroup contentParent = mSubDecor.findViewById(android.R.id.content);
        contentParent.removeAllViews();
        LayoutInflater.from(mContext).inflate(resId, contentParent);
        mAppCompatWindowCallback.getWrapped().onContentChanged();
    }
```

我们的布局最终是在这里加载的。

```
LayoutInflater.from(mContext).inflate(resId, contentParent);
```

但是请注意，这里至是加载，并没有涉及绘制到屏幕上。

那么到底是哪里开始绘制的呢？

还记得上一篇博客中的performLaunchActivity吗？这里面调用了activity的onCreat, 也就是调用了setContentView.

在这个方法之后。

```java
 Activity a = this.performLaunchActivity(r, customIntent);
        if (a != null) {
            r.createdConfig = new Configuration(this.mConfiguration);
            this.reportSizeConfigurations(r);
            Bundle oldState = r.state;
            this.handleResumeActivity(r.token, false, r.isForward, !r.activity.mFinished && !r.startsNotResumed, r.lastProcessedSeq, reason);
```

注意还用了一个handleResumeActivity。

```java
 final void handleResumeActivity(IBinder token, boolean clearHide, boolean isForward, boolean reallyResume, int seq, String reason) {
        ActivityThread.ActivityClientRecord r = (ActivityThread.ActivityClientRecord)this.mActivities.get(token);
        if (checkAndUpdateLifecycleSeq(seq, r, "resumeActivity")) {
            this.unscheduleGcIdler();
            this.mSomeActivitiesChanged = true;
            r = this.performResumeActivity(token, clearHide, reason);
           
              
            
            WindowManager wm;      
            wm = a.getWindowManager();
           ViewRootImpl impl = decor.getViewRootImpl();
               wm.addView(decor, l);
     
            
        }
```



通过前面的流程我门知道，onCreate之行完成之后，所有资源交给WindowManager保管

在这里，将我们的VIew交给了WindowManager,此处调用了addView

![img](https:////upload-images.jianshu.io/upload_images/12625345-a5273f92741d44c4.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)



![img](https:////upload-images.jianshu.io/upload_images/12625345-defb763d0df50d9a.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)



![img](https:////upload-images.jianshu.io/upload_images/12625345-4b74029c69bcf09c.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)



进入addView之后我们发现了一段这样的代码，他将视图，和参数还有我门的一个ViewRoot对象都用了容器去装在了起来，那么在此处我门可以得出，是将所有的相关对象保存起来

mViews保存的是View对象，DecorView

mRoots保存和顶层View关联的ViewRootImpl对象

mParams保存的是创建顶层View的layout参数。

而WindowManagerGlobal类也负责和WMS通信

而在此时，有一句关键代码root.setView，这里是将我们的参数，和视图同时交给了ViewRoot，那么这个时候我们来看下ViewRoot当中的setView干了什么

终于在这里让我发现了让我明白的一步

![img](https:////upload-images.jianshu.io/upload_images/12625345-e2e78bd4adc463a1.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

image

在这里我门会看到view.assignParent的设置是this， 那么也就是说在view当中parent其实实际上是ViewRoot

那么在setContentView当中调用了一个setLayoutParams()是调用的ViewRoot的

而在ViewRoot当中发现了setLayoutParams和preformLayout对requestLayout方法的调用

在requestLayout当中发现了对scheduleTraversals方法的调用而scheduleTraversals当中调用了doTraversal的访问，最终访问到了performTraversals()，而在这个里面，我发现了整体的绘制流程的调用

当前里面依次是用了

![img](https:////upload-images.jianshu.io/upload_images/12625345-d84ba1fb6d93a3bc.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)



![img](https:////upload-images.jianshu.io/upload_images/12625345-57927a7ceb2c573e.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

![img](https:////upload-images.jianshu.io/upload_images/12625345-e765cee0d7a371cc.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)



UI绘制先回去测量布局，然后在进行布局的摆放，当所有的布局测量摆放完毕之后，进行绘制。

 ![img](https://upload-images.jianshu.io/upload_images/12625345-3d011d6ae19cdf86.png?imageMogr2/auto-orient/strip|imageView2/2/w/794/format/webp) 

