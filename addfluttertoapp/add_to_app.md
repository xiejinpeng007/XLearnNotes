# 给现有 App 引入 Flutter Module

最近 Flutter 很火，相信长得帅的人都已经对它都有了初步的了解。  
不过由于目前默认使用 Flutter 作为框架接管整个 App 进行开发，不够灵活：一方面使用纯 Flutter 开发需要在项目开始之前仔细评估是否有难以实现的功能；另一方面现有的 App 想使用 Flutter 的话很难全部转换过去。  
很容易想到在现有的 App 的基础上加入 Flutter 作为部分画面/功能的实现是一个理想的方案，也更有利于做技术尝试和风险控制。
实际上目前 Flutter 官方提供了两种方案用于给现有 App 加入 Flutter Module，另外还有一些第三方的方案，最近我做了一些尝试，分享一些成果。  
需要注意的是， 给现有 App 引入 Flutter Module 的功能还在实验性的阶段, APIs 和工具链处于未稳定阶段,且需要切换到   `master`  分支（不稳定）使用。


## Android
### 创建一个 Flutter module
假设在 `some/path/MyApp` 下是 Android 项目目录

``` shell
cd some/path
flutter create -t module --org com.example flutter_to_app
```
会在 `some/path/flutter_to_app`生成一个 Flutter Module

### 宿主 App 设置
需要在`app/build.gradle`里设置

```
android {
  //...
  compileOptions {
    sourceCompatibility 1.8
    targetCompatibility 1.8
  }
}
```

### 让 App 依赖 Flutter Module
有两种方案，直接依赖源代码和 aar 产物。

#### 1. 依赖生成的 aar

``` shell
cd ~/Documents/Android/flutter_to_app
flutter build aar
```

```
// MyApp/app/build.gradle

android {
  // ...
}

repositories {
  maven {
  //可以使用相对路径或者绝对路径
    url 'some/path/flutter_to_app/build/host/outputs/repo'
  }
}

dependencies {
  // ...
  releaseCompile ('com.example. flutter_to_app:flutter_release:1.0@aar') {
    transitive = true
  }
}
```

可以用 `flutter build aar --debug` 生成 debug 依赖

```
// MyApp/app/build.gradle

dependencies {
  // ...
  debugCompile ('com.example.my_flutter:flutter_debug:1.0@aar') {
    transitive = true
  }
}
```

#### 2.直接依赖源码
依赖 aar 的方式有点麻烦，还需要到 Module 中编译，所以也可以直接依赖源码编译

在宿主 App `settings.gradle`加入

```
// MyApp/settings.gradle
include ':app'
...                                     
setBinding(new Binding([gradle: this]))                                 
evaluate(new File(                                                     
 settingsDir.parentFile,                                                
  'flutter_to_app/.android/include_flutter.groovy'                          
))  
```
上面的`File()`路径是 flutter module 相对 host app 的路径。binding 和 `include_flutter.groovy` 脚本引入 flutter module 本身和相关的 plugin。  

最后，依赖模块:

```
// MyApp/app/build.gradle
dependencies {
  implementation project(':flutter')
}
```


### 在 Android 项目中使用 Flutter Module
目前有两种方式实现，分别在

1. `io.flutter.facade.*`
2. `io.flutter.embedding.android.*`

两个包下， 第一种已经被 deprecated ,第二种还处于 technical preview 阶段，所以两种版本的 API 都还不稳定，但可以大概看一下两种方式。
#### 以前的方式（deprecated） ( io.flutter.facade )
通过使用 `Flutter.createView`:

```
fab.setOnClickListener(new View.OnClickListener() {
  @Override
  public void onClick(View view) {
    View flutterView = Flutter.createView(
      MainActivity.this,
      getLifecycle(),
      "route1"
    );
    FrameLayout.LayoutParams layout = new FrameLayout.LayoutParams(600, 800);
    layout.leftMargin = 100;
    layout.topMargin = 200;
    addContentView(flutterView, layout);
  }
});
```

通过使用 `Flutter.createFragment`:

```
// MyApp/app/src/main/java/some/package/SomeActivity.java
fab.setOnClickListener(new View.OnClickListener() {
  @Override
  public void onClick(View view) {
    FragmentTransaction tx = getSupportFragmentManager().beginTransaction();
    tx.replace(R.id.someContainer, Flutter.createFragment("route1"));
    tx.commit();
  }
});
```

创建`View`和`Fragment`都非常简单，但是实际测试下来，启动 View (FlutterFragment实际上也是通过 createView 来生成视图的)会有启动时间，体验没那么无缝。

### 新的方式（ io.flutter.embedding.android.* ）

#### 通过 FlutterView ( 继承自 FrameLayout )

```
实例化 FlutterView 嵌入 Native
FlutterView flutterView = new FlutterView(this);
FrameLayout frameLayout = findViewById(R.id.framelayout);
frameLayout.addView(flutterView);
//创建一个 FlutterView 就可以了，这个时候还不会渲染。
//调用下面代码后才会渲染
flutterView.attachToFlutterEngine(flutterEngine);
```



#### 通过 FlutterFragment 打开

##### 通过 xml
	
```
	<fragment
    android:id="@+id/flutterfragment"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:name="io.flutter.embedding.android.FlutterFragment"
    />
```
    
##### 直接实例化

```
flutterFragment = new FlutterFragment.createDefault();
```


#### 通过 FlutterActivity 打开

##### 在 AndroidManifest.xml 中注册

```
    <activity
        android:name="io.flutter.embedding.android.FlutterActivity"
        android:theme="@style/LaunchTheme"
        android:configChanges="orientation|keyboardHidden|keyboard|screenSize|locale|layoutDirection|fontScale|screenLayout|density"
        android:hardwareAccelerated="true"
        android:windowSoftInputMode="adjustResize"
        android:exported="true"
        />
```

##### 默认启动方式

```
	//默认路由为 '/'
    Intent defaultFlutter = new FlutterActivity.createDefaultIntent(currentActivity);
    startActivity(defaultFlutter);
```

##### 启动到指定路由

```
    Intent customFlutter = new FlutterActivity.IntentBuilder()
      .initialRoute("someOtherRoute")
      .build(currentActivity);
    startActivity(customFlutter);
```

###  FlutterEngine 缓存机制

实际上，通过 API 和源码可以看出，新版的 Flutter 相关类`io.flutter.embedding.android.*`完全重新设计了 Native 调用的方式，从包名（embedding）就可以看出是希望嵌入 Native， 其中一个重要的变化是加入了 `FlutterEngine` 的缓存机制。
通过老的方式启动 Flutter 的响应时间长包括了需要启动`FlutterEngine`的时间，可以理解为冷启动，而且从原生的不同`Activity / ViewController` 启动 Flutter 都需要启动一个新的 `FlutterEngine`，所以不仅第一次启动 Flutter 时间长 ，每次启动都会需要同样的时间。比如下面的情况

`Native A -> Flutter B -> Native C -> Flutter D`  

这样从`Native A` 和 `Native B`启动时会实例化两个`FlutterEngine`。  

这样不仅慢，对资源的开销也更多。
为了解决这个问题，新的解决方案引入了`FlutterEngine` 缓存机制。


#### 1. 使用 FlutterEngineCache 

```
// 实例化 FlutterEngine.
FlutterEngine flutterEngine = new FlutterEngine(context);

// 预热
 flutterEngine
  .getDartExecutor()
  .executeDartEntrypoint(
    DartEntrypoint.createDefault()
  );
  
 //放入 FlutterEngineCache
  FlutterEngineCache
  .getInstance()
  .put("my_engine_id", flutterEngine);
  
  //启动 Activity 的时候使用
  Intent intent = FlutterActivity
  .withCachedEngine("my_engine_id")
  .build();
  startActivity(intent);
  
  //实例化 Fragment
  FlutterFragment flutterFragment = FlutterFragment
  .withCachedEngine("my_engine_id")
  .build();
```

#### 2. 继承 FlutterFragment / FlutterActivity 

自行处理存储 FlutterEngine 的地方

```
public class MyFlutterFragment extends FlutterFragment {
  @Override
  @Nullable
  protected FlutterEngine provideFlutterEngine(@NonNull Context context) {
    //自行存储 FlutterEngine 实例
    return MyFlutterEngine.getFlutterEngine();

    //比如 Application 中
    return ((MyApp) context.getApplication).getFlutterEngine();
  }
}
```

```
public class MyFlutterActivity extends FlutterActivity {
  @Nullable
  @Override
  public FlutterEngine provideFlutterEngine(@NonNull Context context) {
    FlutterEngine flutterEngine;
    //自行存储 FlutterEngine 实例
    flutterEngine = MyFlutterEngineCache.getFlutterEngine();
    
    //比如 Application 中
    flutterEngine = ((MyApp) getApplication()).getFlutterEngine();

    return flutterEngine;
  }
}
```

#### 3. 在 Activity 实现 FlutterEngineProvider 接口


```
public class MyActivity extends Activity implements FlutterEngineProvider {
  @Override
  @Nullable
  FlutterEngine provideFlutterEngine(@NonNull Context context) {
    //自行存储 FlutterEngine 实例
    return MyFlutterEngine.getFlutterEngine();
    
    //比如 Application 中
    return ((MyApp) context.getApplication).getFlutterEngine();
  }
}
```

### FlutterBoost 方案

> 新一代 Flutter-Native 混合解决方案。 FlutterBoost是一个Flutter插件，它可以轻松地为现有原生应用程序提供Flutter混合集成方案。FlutterBoost的理念是将Flutter像Webview那样来使用。在现有应用程序中同时管理Native页面和Flutter页面并非易事。 FlutterBoost帮你处理页面的映射和跳转，你只需关心页面的名字和参数即可（通常可以是URL）。

FlutterBoost 是闲鱼开源处理 Flutter-Native 混合开发的解决方案，是一个热门的方案，但和官方方案对比我认为有两个重要的异同点：

1. 当时闲鱼设计这个库其中的一个重要目的就是为了解决 FlutterEngine 无法重用的问题（当时 Flutter 团队还没有可以处理 FlutterEngine 重用的方案），而现在 Flutter 团队推出的新的解决方案也可以解决这个问题。
2. 目前 Flutter 官方的方案的细粒度更小，可以通过 View 的方式调用 Flutter ，也就是说你可以只将画面中的某一个图表用 Flutter 替换。


最后，官方的两种方案一种已经被舍弃一种还处于实验性阶段，目前最新方案的`Milestone` 是12月，所以到时候再次评估可行性。而国内大厂基本上各自都有自己的解决方案，所以目前使用官方方案的话还需要仔细评估。

## iOs

*WIP*
