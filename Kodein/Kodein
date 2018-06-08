# Kodein

```kotlin
//BaseFragment
baseFragment extend SupportFragmentInjector{

override val injector: KodeinInjector = KodeinInjector()

override fun onCreate(savedInstanceState: Bundle?){
    initializeInjector()
}

    override fun onDestroy() {
        destroyInjector()
    }
```

```kotlin
//MainApplication

override val kodein: Kodein by Kodein.lazy {
    //导入已经定义的 Module
        import(apiModule)
        import(sharedPrefModule)
        import(autoAndroidModule(this@MainApplication))

    //  绑定:bind() 通过:with() 方式:singleton() 提供实例:{} (with 是个 kotlin 的中辍语法 infix 标记的方法，实质是个方法)
    bind<Context>(AppContextTag) with singleton { this@MainApplication }

}
```

https://github.com/SalomonBrys/Kodein/blob/master/demo-android/src/main/java/kodein/demo/DemoApplication.kt
