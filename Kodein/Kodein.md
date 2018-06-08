## Kotlin 中的依赖注入 KODEIN
用 Java 进行 Android 开发的话，依赖注入这一块一般使用 Dagger ,转用 Kotlin 之后有更多的选择，Kodein 是个不错的库， 4.0 5.0 都使用过了，相对于 Dagger 有很多好处。
* 用 Kotlin 编写利用了更优秀的语言特性 比如类型推断，Dagger 在编写 Component 的时候需要知道注入类的类型
* 无需像 Dagger 一样编写大量模板代码
* 不会像 Dagger 一样在编译期因为其它的编译错误导致无法生成所需文件从而报一堆错。
* ...


### 最基本的使用步骤
1. 在 Application 继承 `KodeinAware` 并绑定依赖
```
class MyApp : Application(), KodeinAware {
	override val kodein = Kodein.lazy { 
	    /* bindings */
	}
}
```
2. 在 context aware 的 Android 类中通过 `closestKodein` 方法获取
3. 加载依赖

```
class MyApp : Application(), KodeinAware {

    //实例化 Application 级的 kodein 通过 DSL 绑定 module
	override val kodein = Kodein.lazy { 
        //导入预设的 android 组件
        import(androidModule(this@MainApplication))
        //绑定或者导入自定义依赖
	}
}
```

### 相关基本概念

#### 在 Application 定义 Application 级的 `Kodein`

```
class MyApp : Application(), KodeinAware {

    //实例化 Application 级的 kodein 通过 DSL 绑定 module
	override val kodein = Kodein.lazy { 
        //导入预设的 android 组件
        import(androidModule(this@MainApplication))
        //绑定或者导入自定义依赖
	}
}
```

#### 通过 closestKodein 恢复 Application 级的 `Kodein`  然后通过 `Kodein` 加载依赖
#####kodein & ds 默认都是懒加载
 
```
class MyActivity : Activity(), KodeinAware {
    override val kodein by closestKodein()
    val ds: DataSource by instance()
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        ds.connect() 
        /* ... */
    }
}
```

#### 使用 Trigger 在 onCreate() 中手动触发加载依赖（取消懒加载）  
同样可以避免依赖死循环（除非加载依赖的方式只有 instance）

```
class MyActivity : Activity(), KodeinAware {
    override val kodein by closestKodein()
    override val kodeinTrigger = KodeinTrigger()
    val ds: DataSource by instance()
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        kodeinTrigger.trigger() 
        /* ... */
    }
}
```

#### 在没有 Non-Context-Aware 的类中加载 Kodein

```
class MyController(androidContext: Context) : KodeinAware {
    override val kodein by androidContext.closestKodein()
    override val kodeinContext = kcontext(androidContext)
    val inflater: LayoutInflater by instance()
}
```

#### 多级 Kodin 依赖
##### 定义 Activity 级的 `Kodein` 继承自 Application 级

```
class MyActivity : Activity(), KodeinAware {
    private val _parentKodein by closestKodein() 
    override val kodein: Kodein = Kodein {
        extend(_parentKodein) 
        /* activity specific bindings */
    }
}
```

#### Activity Retained Kodein
##### 使用 retainedKodein 在 Activity 重启的时候不会重新创建 Kodein

```
class MyActivity : Activity(), KodeinAware {
    private val _parentKodein by closestKodein()
    override val kodein: Kodein by retainedKodein { 
        extend(_parentKodein)
        /* activity specific bindings */
    }
}
```

#### Android scopes 作用域

```
//每个 Activity 一个单例
val kodein = Kodein {
    bind<Controller>() with scoped(androidScope<Activity>()).singleton { ControllerImpl(context) } 
}
```

#### Activity retained scope
#### 同样 activityRetainedScope 在 Activity 重启时不会重新创建依赖

```
val kodein = Kodein {
    bind<Controller>() with scoped(activityRetainedScope).singleton { ControllerImpl() } 
}
```

#### Bindings: Declaring dependencies 申明依赖的一些重要的参数
##### 绑定的方式
* Provider binding : 每次加载都生成新的实例，无参， 传入`() -> T`
* Singleton binding : 单例 传入 `() → T`
* Eager singleton : 单例 创建了 Kodein 实例后立即实例化 传入 `() -> T`
* Factory binding : 每次加载都生成新的实例，需要参数，传入`(A) -> T`
* Multiton binding : 有参的单例，同样参数同样实例，传入 `(A) -> T`
* Tagged bindings : 通过 tag 来区分同类型不同的实例 例如

```
val kodein = Kodein {
    bind<Die>() with ... 
    bind<Die>(tag = "DnD10") with ... 
    bind<Die>(tag = "DnD20") with ... 
}
```

##### 基本上以上的参数涵盖了大部分的通用使用场景，Kodein 还有很多复杂的高级用法

#### JVM: Soft & Weak 两种引用回收机制
1. 使用 WeakReference 在`OutOfMemoryException`之前 JVM 执行 GC
2. 使用 SoftReference 在没有引用的时候就 JVM 执行 GC
```
val kodein = Kodein {
    bind<Map>() with refSingleton(ref = softReference) { WorldMap() } 
    bind<Client>() with refSingleton(ref = weakReference) { id -> clientFromDB(id) } 
}
```

#### Transitive dependencies 依赖传递
##### 依赖中使用依赖的情况，Kotlin 的类型推断系统可以很简单的实现。

```
class Die(private val random: Random, private val sides: Int) {
/*...*/
}

val kodein = Kodein {
    bind<Die>() with singleton { Die(instance(), instance(tag = "max")) } 

    bind<Random>() with provider { SecureRandom() } 
    constant(tag "max") with 5 
}
```

##### 更多高级用法...

* [KODEIN DI: KOtlin DEpendency INjection: 5.0.0](http://kodein.org/Kodein-DI/?5.0)
* [Kodein on Android](http://kodein.org/Kodein-DI/?5.0/android)
* [Getting started](http://kodein.org/Kodein-DI/?5.0/getting-started)
* [Core documentation](http://kodein.org/Kodein-DI/?5.0/core)
* [Sample](https://github.com/Kodein-Framework/Kodein-DI)
