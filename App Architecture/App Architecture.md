
## App Architecture


为什么提出新架构：
  
比如，当你要在自己最喜欢的社交网络app中分享一张照片的时候，你可以想象一下会发生什么。app触发一个camera intent，然后Android OS启动一个camera app来处理这一动作。此时用户已经离开了社交网络的app，但是用户的操作体验却是无缝对接的。而 camera app反过来也可能触发另一个intent，比如启动一个文件选择器，这可能会再次打开另一个app。最后用户回到社交网络app并分享照片。在这期间的任意时刻用户都可被电话打断，打完电话之后继续回来分享照片。

总的来说就是，你的app组件可能是单独启动并且是无序的，而且在任何时候都有可能被系统或者用户销毁。因为app组件生命的短暂性以及生命周期的不可控制性，任何数据都不应该把存放在app组件中，同时app组件之间也不应该相互依赖。

#### => 任何数据都不应该存放在app组件中：

---

#### 通用的架构准则


最重要的一个原则就是尽量在app中做到separation of concerns（关注点分离）。常见的错误就是把所有代码都写在Activity或者Fragment中。任何跟UI和系统交互无关的事情都不应该放在这些类当中。尽可能让它们保持简单轻量可以避免很多生命周期方面的问题。别忘了能并不拥有这些类，它们只是连接app和操作系统的桥梁。根据用户的操作和其它因素，比如低内存，Android OS可能在任何时候销毁它们。为了提供可靠的用户体验，最好把对它们的依赖最小化。  
#### ==> 通过MVVM来分层


第二个很重要的准则是用model驱动UI，最好是持久化的model。之所以要持久化是基于两个原因：如果OS销毁app释放资源，用户数据不会丢失；当网络很差或者断网的时候app可以继续工作。Model是负责app数据处理的组件。它们不依赖于View或者app 组件（Activity，Fragment等），因此它们不会受那些组件的生命周期的影响。保持UI代码的简单，于业务逻辑分离可以让它更易管理。  


![](http://www.jcodecraeer.com/uploads/20170523/1495481828442840.png)



#### 其中比较重要的一点就是为每个层级加入和绑定了生命周期这一个关系。




为了实现这个新的架构，Google提供了几种组件。


### Lifecycle


* Lifecycle 是一个持有组件（比如 activity 或者 fragment）生命周期状态信息的类，并且允许其它对象观察这个状态。  

Lifecycle 主要使用两个枚举来跟踪相关组件的生命周期状态。

* Event:
{ON_ANY,ON_CREATE,ON_DESTROY,ON_PAUSE,ON_RESUME,ON_START,ON_STOP}
* State:
{CREATED,DESTROYED,INITIALIZED,RESUMED,STARTED}

* `LifecycleOwner`
  需要被观察的组件（比如 activity）实现这个接口 => 被观察者
* `LifecycleObserver` 需要观察其它组件生命周期的类 => 观察者




#### 设想这样的情景：


```java
//activity 启动的时候开始位置获取的服务
// 结束的时候暂停位置获取的服务
class MyActivity extends AppCompatActivity {
    private MyLocationListener myLocationListener;
 
    public void onCreate(...) {
        myLocationListener = new MyLocationListener(this, location -> {
            // update UI
        });    }

 
    public void onStart() {
        super.onStart();
        Util.checkUserStatus(result -> {
            //在启动的时候又需要检查一些配置（例如其它的资源准备好没有）
            //如果检查完毕后，activity.stop()，NPE。
            // what if this callback is invoked AFTER activity is stopped?
            if (result) {
                myLocationListener.start();
            }
        });
    }
 
    public void onStop() {
        super.onStop();
        myLocationListener.stop();
    }
}
```

#### 如何使用 Lifecycle 解决
  
在 fragment / activity 这样实现了`LifecycleRegistryOwner`的类，则可以直接注册观察者（也可以自定义）
  
  ```java
  fragment.getLifecycle().addObserver(new MyLocationListener());
  ```
  
如果使用了Lifecycle，则可以让 MyLocationListener 在内部获取组件的生命周期和响应生命周期变化的回调：
  
  ```java
  class MyLocationListener implements LifecycleObserver {
    private boolean enabled = false;
    public MyLocationListener(Context context, Lifecycle lifecycle, Callback callback) {
       ...
    }

    @OnLifecycleEvent(Lifecycle.Event.ON_START)
    void start() {
        if (enabled) {
           // connect
        }
    }

    public void enable() {
        enabled = true;
        if (lifecycle.getState().isAtLeast(STARTED)) {
            // connect if not connected
        }
    }

    @OnLifecycleEvent(Lifecycle.Event.ON_STOP)
    void stop() {
        // disconnect if connected
    }
}
  ```



### LiveData  
LiveData是一个可观察的数据持有者。 无需明确在它与app组件之间创建依赖就可以观察LiveData对象的变化。LiveData还考虑了app组件(activities, fragments, services)的生命周期状态，做了防止对象泄漏的事情。
  
=> 过去是一个普通的POJO类，需要手动管理，现在绑定上了生命周期以及数据变化可以被观察。  
=> 有点类似带有生命周期的Rxjava。


#### 一个普通的LiveData

* ViewModel

```java
public class UserProfileViewModel extends ViewModel {
    ...
    private User user;
    private LiveData<User> user;
    public LiveData<User> getUser() {
        return user;
    }
}
```

* Fragment


```java
    // Update  when the data changes.
@Override
public void onActivityCreated(@Nullable Bundle savedInstanceState) {
    super.onActivityCreated(savedInstanceState);
    viewModel.getUser().observe(this, user -> {
      // update UI
    });
}
```
#### 包装后的LiveData

```java
//现在需要一个可以获取Location数据的类
//并且需要被多处使用

public class LocationLiveData extends LiveData<Location> {
    private static LocationLiveData sInstance;
    private LocationManager locationManager;

    @MainThread
    public static LocationLiveData get(Context context) {
        if (sInstance == null) {
            sInstance = new LocationLiveData(context.getApplicationContext());
        }
        return sInstance;
    }

    private SimpleLocationListener listener = new SimpleLocationListener() {
        @Override
        public void onLocationChanged(Location location) {
            setValue(location);
        }
    };

    private LocationLiveData(Context context) {
        locationManager = (LocationManager) context.getSystemService(
                Context.LOCATION_SERVICE);
    }

    @Override
    protected void onActive() {
        locationManager.requestLocationUpdates(LocationManager.GPS_PROVIDER, 0, 0, listener);
    }

    @Override
    protected void onInactive() {
        locationManager.removeUpdates(listener);
    }
}

```

#### 在fragment中如下使用：
```java
public class MyFragment extends LifecycleFragment {
    public void onActivityCreated (Bundle savedInstanceState) {
        Util.checkUserStatus(result -> {
            if (result) {
                LocationLiveData.get(getActivity()).observe(this, location -> {
                   // update UI
                });
            }
        });
  }
}
```

有三个方法值得注意

* `onActive()`:在LiveData有活跃的观察者的时候调用 => 在上面这个例子中，有观察者激活了需要获取Location数据的时候。
* `onInactive()`:没有活跃的观察者的时候调用 => 在这里可以暂停位置服务的更新
* `setValue()`: 调用此方法通知外部观察者数据改变

这样，在多个fragment中都需要使用位置信息的时候，不仅可以共享资源，LiveData可以自行妥善管理好生命周期。



优点：

* 没有内存泄露
* 不会因为某个activity结束了后,还在使用activity的引用而引起崩溃
* 因为类似屏幕旋转的操作而导致fragment重建或重新进入生命周期，fragment会立即接受到最近的数据。
* 可以共享资源
* 无需手动管理生命周期
* 在fragment.onDestroy()时，就会自动移除观察者 Observer
* 在fragment处于非活动状态时，callback不会触发。  



### ViewModel  
一个ViewModel为特定的UI组件提供数据，比如fragment 或者 activity，并负责和数据处理的业务逻辑部分通信，比如调用其它组件加载数据或者转发用户的修改。ViewModel并不依赖于activity fragment view，也不会被configuration change影响。

```java
public abstract class ViewModel {
    /**
     * This method will be called when this ViewModel is no longer used and will be destroyed.
     * <p>
     * It is useful when ViewModel observes some data and you need to clear this subscription to
     * prevent a leak of this ViewModel.
     */
    @SuppressWarnings("WeakerAccess")
    protected void onCleared() {
    }
}
```

```java

/**
 * Application context aware {@link ViewModel}.
 * <p>
 * Subclasses must have a constructor which accepts {@link Application} as the only parameter.
 * <p>
 */
public class AndroidViewModel extends ViewModel {
    private Application mApplication;

    public AndroidViewModel(Application application) {
        mApplication = application;
    }

    /**
     * Return the application.
     */
    public <T extends Application> T getApplication() {
        return (T) mApplication;
    }
}

```


在fragment中设置 UID，通过fragtment arguments传递，因为ViewModel游离在View的生命周期之外，所以当类似 configuration changed (比如屏幕旋转)的时候会自动将数据保存下来，新的fragment进入生命周期的时候，会收到相同的ViewModel。


```java
public class UserProfileFragment extends LifecycleFragment {
    private static final String UID_KEY = "uid";
    private UserProfileViewModel viewModel;
 
    @Override
    public void onActivityCreated(@Nullable Bundle savedInstanceState) {
        super.onActivityCreated(savedInstanceState);
        String userId = getArguments().getString(UID_KEY);
        viewModel = ViewModelProviders.of(this).get(UserProfileViewModel.class);
        viewModel.init(userId);
    }
 
    @Override
    public View onCreateView(LayoutInflater inflater,
                @Nullable ViewGroup container, @Nullable Bundle savedInstanceState) {
        return inflater.inflate(R.layout.user_profile, container, false);
    }
}
```



![](https://github.com/googlesamples/android-architecture-components/raw/master/BasicSample/docs/images/VM_diagram.png?raw=true)

* 避免了需要重复获取数据从而浪费资源、内存泄露的问题。




### Repository

 类似之前使用的Presenter层，从不同的数据源（内存、网络、硬盘）提供数据，并不需要ViewModel层关心具体操作。
 另外，Google希望在这一层对于数据进行缓存和持久化。

```java
public class UserRepository {
    private Webservice webservice;
    // ...
    public LiveData<User> getUser(int userId) {
        // This is not an optimal implementation, we'll fix it below
        final MutableLiveData<User> data = new MutableLiveData<>();
        webservice.getUser(userId).enqueue(new Callback<User>() {
            @Override
            public void onResponse(Call<User> call, Response<User> response) {
                // error case is left out for brevity
                data.setValue(response.body());
            }
        });
        return data;
    }
}
```

### 管理不同组件间的依赖：
现在 Google 推荐Dagger2


### Room
一个ORM库

#### 参考资料：  
  
* <https://developer.android.com/topic/libraries/architecture/index.html>
*  <https://developer.android.google.cn/topic/libraries/architecture/guide.html>
*  <https://developer.android.google.cn/topic/libraries/architecture/lifecycle.html>
*   <https://developer.android.google.cn/topic/libraries/architecture/viewmodel.html>
*   <https://developer.android.google.cn/topic/libraries/architecture/livedata.html>
*   <https://github.com/googlesamples/android-architecture-components>
*   <http://www.jcodecraeer.com/a/anzhuokaifa/androidkaifa/2017/0523/7963.html>
*   <http://www.jcodecraeer.com/a/anzhuokaifa/androidkaifa/2017/0524/7969.html>
  