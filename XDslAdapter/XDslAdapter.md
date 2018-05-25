Android 开发要写一大堆 Adapter，使用 Kotlin 的话可以通过 Dsl 来完成。  
ViewPager 还没有看到满意的，于是撸了一个针对 ViewPager + DataBinding 的 DslAdapter。

一个用于app首次安装时的教程的 sample code:

```kotlin
       binding.guideViewPager.run {
            xDslPagerAdapter {

                item(R.layout.pager_guide) {
                    model(BR.model to Page(0, R.drawable.img_tutorial01, "下一页"))
                    handle(BR.click to { _: Page -> currentItem = 1 })
                    action { binding ->
                        (binding as? PagerGuideBinding)?.run {
                            (nextPageButton.layoutParams as? ViewGroup.MarginLayoutParams)?.run {
                                marginStart = dp2px(30F)
                                marginEnd = dp2px(30F)
                            }
                        }
                    }
                }

                item(R.layout.pager_guide) {
                    model(BR.model to Page(1, R.drawable.img_tutorial02, "下一页"))
                    handle(BR.click to { _: Page -> currentItem = 2 })
                }

                item(R.layout.pager_guide) {
                    model(BR.model to Page(2, R.drawable.img_tutorial03, "下一页"))
                    handle(BR.click to { _: Page -> currentItem = 3 })
                }

                item(R.layout.pager_guide) {
                    model(BR.model to Page(3, R.drawable.img_tutorial04, "开始使用"))
                    handle(BR.click to { _: Page ->
                        SharedPrefModel.isFistTime = false
                        startActivity(Intent(context, MainActivity::class.java))
                        finish()
                    })
                }
            }
        }
```

简单来说，通过 ViewPager.xDslPagerAdapter 这个拓展方法开始构建 Adapter ，item 中 model 绑定 vm ，handle 绑定响应动作，有更细的处理在 action 中完成。