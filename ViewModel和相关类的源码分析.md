##                                                                                               ViewModel和相关类的源码分析  

 首先我们看我们创建ViewModel的时候所使用的方法

```java
  public static ViewModelProvider of(@NonNull FragmentActivity activity,
            @Nullable Factory factory) {
        Application application = checkApplication(activity);
        if (factory == null) {
            factory = ViewModelProvider.AndroidViewModelFactory.getInstance(application);
        }
        return new ViewModelProvider(activity.getViewModelStore(), factory);
    }

```

首先传入一个activty,通过activty得到了application,大多数时候我们传入的第二个参数都为null,那么这里就创建了一个factory,我们来看看这是什么

```java
 public static class AndroidViewModelFactory extends ViewModelProvider.NewInstanceFactory {

        private static AndroidViewModelFactory sInstance;

        /**
         * Retrieve a singleton instance of AndroidViewModelFactory.
         *
         * @param application an application to pass in {@link AndroidViewModel}
         * @return A valid {@link AndroidViewModelFactory}
         */
        @NonNull
        public static AndroidViewModelFactory getInstance(@NonNull Application application) {
            if (sInstance == null) {
                sInstance = new AndroidViewModelFactory(application);
            }
            return sInstance;
        }

        private Application mApplication;

        /**
         * Creates a {@code AndroidViewModelFactory}
         *
         * @param application an application to pass in {@link AndroidViewModel}
         */
        public AndroidViewModelFactory(@NonNull Application application) {
            mApplication = application;
        }

        @NonNull
        @Override
        public <T extends ViewModel> T create(@NonNull Class<T> modelClass) {
            if (AndroidViewModel.class.isAssignableFrom(modelClass)) {
                //noinspection TryWithIdenticalCatches
                try {
                    return modelClass.getConstructor(Application.class).newInstance(mApplication);
                } catch (NoSuchMethodException e) {
                    throw new RuntimeException("Cannot create an instance of " + modelClass, e);
                } catch (IllegalAccessException e) {
                    throw new RuntimeException("Cannot create an instance of " + modelClass, e);
                } catch (InstantiationException e) {
                    throw new RuntimeException("Cannot create an instance of " + modelClass, e);
                } catch (InvocationTargetException e) {
                    throw new RuntimeException("Cannot create an instance of " + modelClass, e);
                }
            }
            return super.create(modelClass);
        }
    }
```

这个类是在ViewModelProvider中的一个静态内部类，静态方法getInstance返回一个AndroidViewModelFactory单例，其构造方法是传入一个Application，然后重点是最后一个方法create，这个方法是重写的，看其方法体，可以大致猜到这是创建ViewModel的方法，最后调用了父类的create方法，我们看看其父类ViewModelProvider.NewInstanceFactory

```java
 public static class NewInstanceFactory implements Factory {

        @SuppressWarnings("ClassNewInstance")
        @NonNull
        @Override
        public <T extends ViewModel> T create(@NonNull Class<T> modelClass) {
            //noinspection TryWithIdenticalCatches
            try {
                //这里是重点，通过反射拿到传入的ViewModel的类的对象 
                return modelClass.newInstance();
            } catch (InstantiationException e) {
                throw new RuntimeException("Cannot create an instance of " + modelClass, e);
            } catch (IllegalAccessException e) {
                throw new RuntimeException("Cannot create an instance of " + modelClass, e);
            }
        }
    }
```



```java
   public interface Factory {
        /**
         * Creates a new instance of the given {@code Class}.
         * <p>								zz	
         *
         * @param modelClass a {@code Class} whose instance is requested
         * @param <T>        The type parameter for the ViewModel.
         * @return a newly created ViewModel
         */
        @NonNull
        <T extends ViewModel> T create(@NonNull Class<T> modelClass);
    }
```

很显然了，create方法就是通过反射返回ViewModle的对象。

到此为止，factory的部分就分析完了，最后一定是调用了factory的create方法创建ViewModle对象的。

然后我们回到最初的时候

```java
 public static ViewModelProvider of(@NonNull FragmentActivity activity,
            @Nullable Factory factory) {
        Application application = checkApplication(activity);
        if (factory == null) {
            factory = ViewModelProvider.AndroidViewModelFactory.getInstance(application);
        }
        return new ViewModelProvider(activity.getViewModelStore(), factory);
    }
```

返回的并不是factory而是ViewModelProvieder

看看这个类



```java
   
   public class ViewModelProvider {
      private final Factory mFactory;
    private final ViewModelStore mViewModelStore;
   public ViewModelProvider(@NonNull ViewModelStore store, @NonNull Factory factory) {
        mFactory = factory;
        mViewModelStore = store;
    }
    public <T extends ViewModel> T get(@NonNull Class<T> modelClass) {
        String canonicalName = modelClass.getCanonicalName();
        if (canonicalName == null) {
            throw new IllegalArgumentException("Local and anonymous classes can not be ViewModels");
        }
        return get(DEFAULT_KEY + ":" + canonicalName, modelClass);
    }
   
   
    @NonNull
    @MainThread
    public <T extends ViewModel> T get(@NonNull String key, @NonNull Class<T> modelClass) {
        ViewModel viewModel = mViewModelStore.get(key);

        if (modelClass.isInstance(viewModel)) {
            //noinspection unchecked
            return (T) viewModel;
        } else {
            //noinspection StatementWithEmptyBody
            if (viewModel != null) {
                // TODO: log a warning.
            }
        }
        if (mFactory instanceof KeyedFactory) {
            viewModel = ((KeyedFactory) (mFactory)).create(key, modelClass);
        } else {
            viewModel = (mFactory).create(modelClass);
        }
        mViewModelStore.put(key, viewModel);
        //noinspection unchecked
        return (T) viewModel;
    }

   
   
   }
       
```

可见其构造方法传入了一个ViewModelStore，这又是啥

```java

/**
 * Class to store {@code ViewModels}.
 * <p>
 * An instance of {@code ViewModelStore} must be retained through configuration changes:
 * if an owner of this {@code ViewModelStore} is destroyed and recreated due to configuration
 * changes, new instance of an owner should still have the same old instance of
 * {@code ViewModelStore}.
 * <p>
 * If an owner of this {@code ViewModelStore} is destroyed and is not going to be recreated,
 * then it should call {@link #clear()} on this {@code ViewModelStore}, so {@code ViewModels} would
 * be notified that they are no longer used.
 * <p>
 * Use {@link ViewModelStoreOwner#getViewModelStore()} to retrieve a {@code ViewModelStore} for
 * activities and fragments.
 */

/**
*翻译一下注释，一个ViewModelStore的实例必须被保持其实配置变化，如果ViewModelStore的拥有者由于配置改变被销毁了
*，那么新的owner（被重新创建的owner）必须拥有相同的ViewModelStore实例，如果ViewModelStore的拥有者被销毁了并且不会再被创建，那么就因该调用clear方法，提醒所有的ViewModels他们已经不会再被用到了
用getViewModelStore()可以得到activities 或者 fragments 的ViewModelStore。

*/


public class ViewModelStore {

    private final HashMap<String, ViewModel> mMap = new HashMap<>();

    final void put(String key, ViewModel viewModel) {
        ViewModel oldViewModel = mMap.put(key, viewModel);
        if (oldViewModel != null) {
            oldViewModel.onCleared();
        }
    }

    final ViewModel get(String key) {
        return mMap.get(key);
    }

    Set<String> keys() {
        return new HashSet<>(mMap.keySet());
    }

    /**
     *  Clears internal storage and notifies ViewModels that they are no longer used.
     */
    public final void clear() {
        for (ViewModel vm : mMap.values()) {
            vm.clear();
        }
        mMap.clear();
    }
}

```

翻译一下注释：

一个ViewModelStore的实例必须被保持其实配置变化，如果ViewModelStore的拥有者由于配置改变被销毁了
，那么新的owner（被重新创建的owner）必须拥有相同的ViewModelStore实例，如果ViewModelStore的拥有者被销毁了并且不会再被创建，那么就因该调用clear方法，提醒所有的ViewModels他们已经不会再被用到了
用getViewModelStore()可以得到activities 或者 fragments 的ViewModelStore。

记得谷歌推出了ViewModel的一个重要的作用吗？

那就是即使acitivity因为配置改变而被重建（比如屏幕旋转），它的viewModel也不会改变，因为数据存在viewModel里面，所以数据不会丢失。

这个功能就和这个类有巨大的关系，我们来一探究性。

ViewModelStore的设计很简单，维护了一个HashMap<String, ViewModel>，这很显然就是来存储viewModel的。

那么到底是怎样实现activity销毁了但是ViewModelStore却没有被销毁呢？我们想一下，我们以前常常头疼的内存泄露，是不是这个对象本应该被回收了，但是只要一个东西引用着它，那么就回收不了，那么这里是不是这一样的思想呢？

对的，如果你去看稍微老一点的讲解viewModel的博客的话，会说实现原理是设置fragment的一个属性，让fragment即使activity销毁重建的时候仍然保存，然后fragment里面存一个viewModel的Map,这个就是ViewModelStore的前身，然后让这个不可见fragment被activty托管，再把acitivy对应的viewModel加入到fragment的map中。这样就可以保存下来了。

但是如今的viewModel早已去除了这种实现。但是思路是一样的，用一个生命周期更长的托管viewModelStore.

这里就是我们的NonConfigurationInstances了

我们来看看

```java
 @NonNull
    @Override
    public ViewModelStore getViewModelStore() {
        if (getApplication() == null) {
            throw new IllegalStateException("Your activity is not yet attached to the "
                    + "Application instance. You can't request ViewModel before onCreate call.");
        }
        if (mViewModelStore == null) {
            NonConfigurationInstances nc =
                    (NonConfigurationInstances) getLastNonConfigurationInstance();
            if (nc != null) {
                // Restore the ViewModelStore from NonConfigurationInstances
                mViewModelStore = nc.viewModelStore;
            }
            if (mViewModelStore == null) {
                mViewModelStore = new ViewModelStore();
            }
        }
        return mViewModelStore;
    }

```

```java
    static final class NonConfigurationInstances {
        Object custom;
        ViewModelStore viewModelStore;
    }
```



 // Restore the ViewModelStore from NonConfigurationInstances

这句注释说明了一切把ViewModelStore存在NonConfigurationInstances中。这就是核心实现了。至于这个对象为什么在activity重建销毁的时候为什么存活，这就要从ActivityThread说起了，这没必要了。



然后提供了put方法,这其中调用了viewModel的方法，所以让让我们来看看我们的ViewMoldel类

```java
/**
 * ViewModel is a class that is responsible for preparing and managing the data for
 * an {@link android.app.Activity Activity} or a {@link androidx.fragment.app.Fragment Fragment}.
 * It also handles the communication of the Activity / Fragment with the rest of the application
 * (e.g. calling the business logic classes).
 * <p>
 * A ViewModel is always created in association with a scope (an fragment or an activity) and will
 * be retained as long as the scope is alive. E.g. if it is an Activity, until it is
 * finished.
 * <p>
 * In other words, this means that a ViewModel will not be destroyed if its owner is destroyed for a
 * configuration change (e.g. rotation). The new instance of the owner will just re-connected to the
 * existing ViewModel.
 * <p>
 * The purpose of the ViewModel is to acquire and keep the information that is necessary for an
 * Activity or a Fragment. The Activity or the Fragment should be able to observe changes in the
 * ViewModel. ViewModels usually expose this information via {@link LiveData} or Android Data
 * Binding. You can also use any observability construct from you favorite framework.
 * <p>
 * ViewModel's only responsibility is to manage the data for the UI. It <b>should never</b> access
 * your view hierarchy or hold a reference back to the Activity or the Fragment.
 * <p>
 * Typical usage from an Activity standpoint would be:
 * <pre>
 * public class UserActivity extends Activity {
 *
 *     {@literal @}Override
 *     protected void onCreate(Bundle savedInstanceState) {
 *         super.onCreate(savedInstanceState);
 *         setContentView(R.layout.user_activity_layout);
 *         final UserModel viewModel = ViewModelProviders.of(this).get(UserModel.class);
 *         viewModel.userLiveData.observer(this, new Observer<User>() {
 *            {@literal @}Override
 *             public void onChanged(@Nullable User data) {
 *                 // update ui.
 *             }
 *         });
 *         findViewById(R.id.button).setOnClickListener(new View.OnClickListener() {
 *             {@literal @}Override
 *             public void onClick(View v) {
 *                  viewModel.doAction();
 *             }
 *         });
 *     }
 * }
 * </pre>
 *
 * ViewModel would be:
 * <pre>
 * public class UserModel extends ViewModel {
 *     private final MutableLiveData&lt;User&gt; userLiveData = new MutableLiveData&lt;&gt;();
 *
 *     public LiveData&lt;User&gt; getUser() {
 *         return userLiveData;
 *     }
 *
 *     public UserModel() {
 *         // trigger user load.
 *     }
 *
 *     void doAction() {
 *         // depending on the action, do necessary business logic calls and update the
 *         // userLiveData.
 *     }
 * }
 * </pre>
 *
 * <p>
 * ViewModels can also be used as a communication layer between different Fragments of an Activity.
 * Each Fragment can acquire the ViewModel using the same key via their Activity. This allows
 * communication between Fragments in a de-coupled fashion such that they never need to talk to
 * the other Fragment directly.
 * <pre>
 * public class MyFragment extends Fragment {
 *     public void onStart() {
 *         UserModel userModel = ViewModelProviders.of(getActivity()).get(UserModel.class);
 *     }
 * }
 * </pre>
 * </>
 */
public abstract class ViewModel {
    @Nullable
    private final ConcurrentHashMap<String, Object> mBagOfTags = new ConcurrentHashMap<>();
    private volatile boolean mCleared = false;

    /**
     * This method will be called when this ViewModel is no longer used and will be destroyed.
     * <p>
     * It is useful when ViewModel observes some data and you need to clear this subscription to
     * prevent a leak of this ViewModel.
     */
    @SuppressWarnings("WeakerAccess")
    protected void onCleared() {
    }

    @MainThread
    final void clear() {
        mCleared = true;
        // Since clear() is final, this method is still called on mock objects
        // and in those cases, mBagOfTags is null. It'll always be empty though
        // because setTagIfAbsent and getTag are not final so we can skip
        // clearing it
        if (mBagOfTags != null) {
            for (Object value : mBagOfTags.values()) {
                // see comment for the similar call in setTagIfAbsent
                closeWithRuntimeException(value);
            }
        }
        onCleared();
    }

    /**
     * Sets a tag associated with this viewmodel and a key.
     * If the given {@code newValue} is {@link Closeable},
     * it will be closed once {@link #clear()}.
     * <p>
     * If a value was already set for the given key, this calls do nothing and
     * returns currently associated value, the given {@code newValue} would be ignored
     * <p>
     * If the ViewModel was already cleared then close() would be called on the returned object if
     * it implements {@link Closeable}. The same object may receive multiple close calls, so method
     * should be idempotent.
     */
    <T> T setTagIfAbsent(String key, T newValue) {
        @SuppressWarnings("unchecked") T previous = (T) mBagOfTags.putIfAbsent(key, newValue);
        T result = previous == null ? newValue : previous;
        if (mCleared) {
            // It is possible that we'll call close() multiple times on the same object, but
            // Closeable interface requires close method to be idempotent:
            // "if the stream is already closed then invoking this method has no effect." (c)
            closeWithRuntimeException(result);
        }
        return result;
    }

    /**
     * Returns the tag associated with this viewmodel and the specified key.
     */
    @SuppressWarnings("TypeParameterUnusedInFormals")
    <T> T getTag(String key) {
        //noinspection unchecked
        return (T) mBagOfTags.get(key);
    }

    private static void closeWithRuntimeException(Object obj) {
        if (obj instanceof Closeable) {
            try {
                ((Closeable) obj).close();
            } catch (IOException e) {
                throw new RuntimeException(e);
            }
        }
    }
}

```

简单的说下顶上注释的意思，viewModel总是和一个actvity或者fragment向关联，只有当actvity或者fragment真正被销毁不再创建的时候它才会被销毁，activity或者fragment需要观察viewModel中的数据变化，viewModel也应该暴露一些信息通过liveData或者DataBinding,viewModel的唯一作用就是为了UI而管理数据，它不能与获取view阶层，也不能有fragment或者Activity的引用。viewModel还有一个功能就是可以用来同一个activity托管的fragment之间的通信，因为通过同一个activity可以得到同一个viewModel.

主要看clear方法，清除hashmap里面的所有引用，并且执行onClear方法，这个方法是可以重写的，It is useful when ViewModel observes some data and you need to clear this subscription to

     * prevent a leak of this ViewModel.









