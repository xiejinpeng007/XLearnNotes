>从 2021 年 8 月起，新应用需要使用 Android App Bundle 才能在 Google Play 中发布。现在，Play Feature Delivery 或 Play Asset Delivery 支持大小超过 150 MB 的新应用。

由于 Google Play 的政策要求，最近越来多的项目在调查 AppBundle 的问题，总结一下。

### App Bundle 是什么

Android App Bundle 是一种发布格式，其中包含应用的所有经过编译的代码和资源，它会将 APK 生成及签名交由 Google Play 来完成。
Google 主要想通过 App Bundle 来实现优化 APK 体积、动态分发等功能，方式就是 `Split APKs(拆分apk)`。

从下图可以看到, 资源被分布到了各个模块中，每个模块的组织方式都和 APK 相似，是因为最终每个模块都可以作为单独的 APK 生成。

![aab_format-2x.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/38cfed3e911a43d89c6b9376fbd5a421~tplv-k3u1fbpfcp-watermark.image)

图1 的 AppBundle 包含了一个基本模块、两个功能模块和两个资源包。

更详细的目录结构：

* base/、feature1/ 和 feature2/：其中每个顶级目录都表示一个不同的应用模块。应用的基本模块始终包含在 App Bundle 的 base 目录中。不过，为每个功能模块的目录提供的名称由模块清单中的 split 属性指定。如需了解详情，请参阅功能模块清单。

* asset_pack_1/ 和 asset_pack_2/：对于需要大量图形处理的大型应用或游戏，您可以将资产模块化处理为资源包。资源包因体积上限较高而成为游戏的理想之选。您可以按照三种分发模式（即，安装时分发、快速跟进式分发和按需分发）自定义如何以及何时将各个资源包下载到设备上。所有资源包都在 Google Play 上托管并从 Google Play 提供。如需详细了解如何将资源包添加到您的 app bundle。

* BUNDLE-METADATA/：此目录包含元数据文件，其中包含对工具或应用商店有用的信息。此类元数据文件可能包含 ProGuard 映射和应用的 DEX 文件的完整列表。此目录中的文件未打包到您应用的 APK 中。

* 模块协议缓冲区 (*.pb) 文件：这些文件提供了一些元数据，有助于向各个应用商店（如 Google Play）说明每个应用模块的内容。例如，BundleConfig.pb 提供了有关 bundle 本身的信息（如用于构建 app bundle 的构建工具版本），native.pb 和 resources.pb 说明了每个模块中的代码和资源，这在 Google Play 针对不同的设备配置优化 APK 时非常有用。

* manifest/：与 APK 不同，app bundle 将每个模块的 AndroidManifest.xml 文件存储在这个单独的目录中。

* dex/：与 APK 不同，app bundle 将每个模块的 DEX 文件存储在这个单独的目录中。

* res/、lib/ 和 assets/：这些目录与典型 APK 中的目录完全相同。当您上传 App Bundle 时，Google Play 会检查这些目录并且仅打包满足目标设备配置需求的文件，同时保留文件路径。

* root/：此目录存储的文件之后会重新定位到包含此目录所在模块的任意 APK 的根目录。例如，app bundle 的 base/root/ 目录可能包含您的应用使用 Class.getResource() 加载的基于 Java 的资源。这些文件之后会重新定位到您应用的基本 APK 和 Google Play 生成的每个多 APK 的根目录。此目录中的路径也会保留下来。也就是说，目录（及其子目录）也会重新定位到 APK 的根目录。

###  Split APKs （拆分 apk）

AppBundle 按以上方式组织资源之后，就是为了便于拆分 APK 。
拆分 APK 可以从两个维度来考虑：

#### 1. 根据机型所需的资源拆分成不同的 apk

由于 Android 手机类型很多需要适配不同的资源：`ABI (armeabi armeabi-v7a arm64-v8a x86..)` `屏幕密度` `语言`。最后各种资源都会被打包到 apk 里。

![appbundle_apk-2.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/199a273660974e16a0f5cb4c44b324bc~tplv-k3u1fbpfcp-watermark.image)

以上面 apk 里的 `res` `lib`文件夹为例可以看到，过去我们打包的 apk 都包含了以上所有的资源，而且 so 库 + 图片布局资源 的体积在整个 apk 中占很大的比例。

但实际上每一台手机从各个维度来讲都只需要一种对应的资源：例如 cpu 是 armeabi-v7a 的就只需要下载 armeabi-v7a 的 so 库；屏幕 PPI 是 400 就只需要下载 xxxhdpi 的图片布局资源 ；中文语言使用者只需要下载中文语言资源。  
参考下图：

![appbundlesplit.gif](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/edea89b9fa634229b02bacdf95bf3639~tplv-k3u1fbpfcp-watermark.image)

#### 2. 根据基本功能、附加功能拆分成不同的 apk

app 初次下载只需要下载基本功能，需要用到的时候再进行附加功能模块下载。
    
根据上面的两个维度的考虑，Google Play 会把 AppBundle 打包成三类 APK

* 基本 APK：  
此 APK 中包含了所有其他拆分 APK 都可以访问的代码和资源，并提供应用的基本功能。当用户请求下载您的应用时，会首先下载并安装该 APK。这是因为只有基本 APK 的清单才包含关于应用的服务、内容提供方、权限、平台版本要求和对系统功能的依赖性的完整声明。Google Play 会根据项目的应用模块（即基本模块）为应用生成基本 APK。如果您想减小应用的初始下载大小，请一定要注意，此模块中包含的所有代码和资源都包含在应用的基本 APK 中。

* 配置 APK：  
每个配置 APK 都包含针对特定屏幕密度、CPU 架构或语言的原生库和资源。当用户下载您的应用时，他们的设备只会下载并安装该设备对应的配置 APK。每个配置 APK 都是基本 APK 或功能模块 APK 的依赖项。也就是说，配置 APK 会随它们为之提供代码和资源的 APK 一起下载和安装。与基本模块和功能模块不同，您不需要为配置 APK 单独创建模块。如果您在为基本模块和功能模块组织管理配置专用的备用资源时遵循了标准实践，Google Play 会自动为您生成配置 APK。

* 功能模块 APK：  
每个功能模块 APK 都包含您使用功能模块进行了模块化处理的某项应用功能的代码和资源。您随后可以自定义如何以及何时将该功能下载到设备上。例如，使用 Play 核心库，可在将基本 APK 安装到设备上之后再按需安装某些功能，以向用户提供额外的功能。假设我们有一款聊天应用，它仅在用户想要拍摄并发送照片时才下载并安装该功能。由于功能模块在安装时可能不可用，因此您应将所有通用代码和资源包含在基本 APK 中。也就是说，您的功能模块应假定在安装时只有基本 APK 的代码和资源可用。Google Play 会根据项目的功能模块为应用生成功能模块 APK。

一个包含三个功能模块并支持多种设备配置的应用如下的 APK 之间依赖关系树如下：


![apk_splits_tree-2x.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/64eb79d3e6b5445396c34cf442b514de~tplv-k3u1fbpfcp-watermark.image)

###  构建、测试、配置 AppBundle 

#### 构建 AppBundle

和以前打包 APK 类似，命令 assemble 换成 bundle 就可以了。如

```
./gradlew assembleProductionRelease
 ->  
./gradlew bundleProductionRelease
```

#### 测试 AppBundle

*  通过 Google Play 的测试频道部署 AppBundle
      比较麻烦，需要比较正式的配置
      
*    通过 Google Play 的内部分享频道部署 AppBundle
      交付测试人员测试的话推荐这种方式，可以直接上传 .aab 分享下载，比测试频道的限制更少：
      App 签名和 versionCode 这些都没有限制。  
      参考下图

![internalshare-2.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c4eca29b074c4d9695671d4022ae1a57~tplv-k3u1fbpfcp-watermark.image)
      
*  使用命令行工具`bundletool`
    1. 生成 apks 
    
    ```
    //默认使用debug的签名
    bundletool build-apks --bundle=/MyApp/my_app.aab --output=/MyApp/my_app.apks
    //或指定签名
    bundletool build-apks --bundle=/MyApp/my_app.aab --output=/MyApp/my_app.apks
    --ks=/MyApp/keystore.jks
    --ks-pass=file:/MyApp/keystore.pwd
    --ks-key-alias=MyKeyAlias
    --key-pass=file:/MyApp/key.pwd   
    ```
    
    2. 从 apks 部署 apk 到设备 
    
    ```
    bundletool install-apks --apks=/MyApp/my_app.apks
    ```
##### 生成 apks 时的一些配置

 默认生成 apks 时，所有相关的配置集都会打包到 apk，测试的时候一般只需要特定机型的 apk，有几种配置的方法：

 * 只生成已连接到 adb 机型的 apk

 ```
 bundletool build-apks --connected-device
--bundle=/MyApp/my_app.aab --output=/MyApp/my_app.apks
 ```

 * 生成一个描述机型配置的 JSON 文件

 ```
 bundletool get-device-spec --output=/tmp/device-spec.json
 ```

 然后生成 apks 的时候配置一下

 ```
 bundletool build-apks --device-spec=/MyApp/pixel2.json
--bundle=/MyApp/my_app.aab --output=/MyApp/my_app.apks
 ```

 也可以手动生成或者调整机型配置的 JSON 文件

 ```
 {
  "supportedAbis": ["arm64-v8a", "armeabi-v7a"],
  "supportedLocales": ["en", "fr"],
  "screenDensity": 640,
  "sdkVersion": 27
}
 ```

 其它参数配置参考[文档](https://developer.android.com/studio/command-line/bundletool#generate_apks)
    
##### 从 APKs 中提取指定的 APK

除了只生成特定机型 APK 的APKs之外，也可以从 APKs 中提取指定的 APK，方便安装和估算 APK 大小之类的。

提取指定 apk:

```
bundletool extract-apks
--apks=/MyApp/my_existing_APK_set.apks
--output-dir=/MyApp/my_pixel2_APK_set.apks
--device-spec=/MyApp/bundletool/pixel2.json
```

估算 APK 安装到设备后大小范围 (也可手动指定配置参考[文档](https://developer.android.com/studio/command-line/bundletool#measure_size))

```
bundletool get-size total --apks=/MyApp/my_app.apks
```


### AppBundle 原理和实际效果

 AppBundle 技术最大的不同就是以前只打一个 APK 直接安装 ，现在打包成一个 AppBundle 再打包成 APKs，根据设备配置进行安装，下面实际来测试一下有什么不同。

Android 开发人员应该都知道安装的 apk 实际上被放在了一个 APP 的私有的目录(/data/app)。 
以 Demo 项目为例，直接打包 apk 后安装到模拟器:


![apk-size-2.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6c97d560add9479ca305d0ef63284c4f~tplv-k3u1fbpfcp-watermark.image)
    
  可以看到安装目录下只有一个 `base.apk` ,大小是 23.5MB
    然后我再打包一个 appbundle ，用 bundletool 生成 APKs 后查看内容
    

![apks.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e44d949881ca4102abf4bdf85dc4a85f~tplv-k3u1fbpfcp-watermark.image)

   可以看到中有很多根据不同维度生成的 APK，实际上 APKs 就是包含所有 split apk 的集合。

   下一步用命令从 APKs 从安装到模拟器后，查看安装目录。


![appbundle-size-2.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/391a2633f9c0456d8fa2fdb75e04d0ae~tplv-k3u1fbpfcp-watermark.image)

   可以看到安装目录里的内容不一样了，被拆分成多个 apk 进行安装。  
   有 base.apk split_config.en.apk split_config.x86.apk split_config.xxhdpi.apk 以及一些其他的如 so库和 dex 文件。

   * base.apk  : 基础的 APK
   * split_config.en.apk 英语语言的资源
   * split_config.x86.apk x86架构的配置
   * split_config.xxhdpi.apk 该设备仅需的 xxhdpi
   * lib/x86 x86架构下用到的 so库
   * oat/x86/base.art base.odex base.vdex 对应以前的 dex 文件

   再对比一下直接安装 apk 和经过 AppBundle 安装拆分后的总大小差异


![size-diff-2.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6b28766022854d14859000a9f9d0cf63~tplv-k3u1fbpfcp-watermark.image)

可以看到两者有 16.5MB 和 (7.5MB-14.8MB) 的差异（意思是根据机型不同安装的 apk 大小范围）对于这个so库和图片资源用的不多的 APP 体积最多可以减少50%+，其它 APP 差距可能会更大，所以效果还是很明显的，并且下载和安装到设备后的体积都能得到优化。

##### 动态交付（Play Feature Delivery）：

除此之外，拆分 apk 还可以实现动态交付的功能，首次安装只下载基本的 apk，非必要的模块动态下载。（依赖于 Google Play 商店）
这个我还没有实际用过，可以参考文档 [Play Feature Delivery](https://developer.android.com/guide/app-bundle/dynamic-delivery)

##### AppBundle 的限制

* Android 5.0

Android 5.0 以上才支持安装拆分 APK，所以 5.0 以下相关的特性都没办法完全利用，不过开发者也不需要太关心， Google Play 会生成相应可以使用的 APK。

* oob 格式

   游戏常用的 oob 格式不再支持，推荐改为 asset_pack

##### AppBundle 签名替换

部分项目的签名流程是在我们签名 apk 后，客户会重新签名再上传到 Google Play：
AppBundle 格式不能用 apksinger 签名，可以改用 jarsigner。
另外一个方法参考后面会提到的 Play 签名计划

####  Play 应用签名计划

Google Play 一直有一个 Play 应用签名计划，以前不用管它，但使用了 AppBundle 的话强制要求使用。一个原因是我们上传的 AppBundle 后，Google Play 需要打包成 APKs ，所以我们需要把签名交给 Google Play.

**Play 应用签名计划会使用两个密钥：应用签名密钥和上传密钥，上传密钥是用于验证我们上传给 Google的 .aab/.apk 的身份，应用签名密钥可以前一样给App签名的。**

原理参考：

![appsigning_googleplayappsigningdiagram_2x.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2d8da13e668047fea2036287aba624d0~tplv-k3u1fbpfcp-watermark.image)

其中关于应用签名密钥有两种选择：

1.  上传加密后的密钥给 Google Play ，然后 Google Play 用这个密钥签名。  
    
2. 上传加密后的密钥给 Google Play，但是 Google Play 会自己生成另一套密钥来给 APK 签名，我们上传的密钥只作为验证 AppBundle 使用。  

    * 好处：万一我们的密钥丢失或被盗了，只需证明身份，那么还可以继续使用 Google Play 生成的签名，方式一的话，就只能更换签名。
    * 坏处：  
    		1. 如果我们想在其它商店或者官网发布有自己密钥签名的 APK，那么就由于和 Google Play 的签名不一样，不能覆盖安装。  
      		2. 一些 SDK 如我们用过的乐天 IDSDK 就需要根据证书指纹认证身份，需要更换为真实签名的指纹。


上传密钥需要用 PEPK 工具加密后上传：


![pepk.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/406e76ceab564424b1b426672466effa~tplv-k3u1fbpfcp-watermark.image)

此外，刚才提到的替换签名的需求也可以通过区分上传密钥和签名来实现，将我们这边打包的 appbundle 的 密钥设置为Play签名计划里的上传密钥，将客户实际打包的密钥设置为应用签名密钥，就可以实现原来重新签名的需求。

### 总结

总的来说，如果我们的 APP工程结构标准，没有用太多的黑科技的话（特殊情况可能会打包失败之类，需要专门排查），通过 AppBundle 可以很轻松地以很低的成本就获得较大的 APK 体积压缩优化，还是很划算的。并且 Google Play 强制要求了，也不用纠结用不用了。项目已经使用 AppBundle 格式一段时间了，没有遇到太大问题。
另外，通过 AppBundle 我认为以后开发 APP 时可以有一些拆分 Module 的想法。这样可以尝试 Play Feature Delivery 、 Instant App （免安装App）这些功能。

参考：  
[https://developer.android.com/guide/app-bundle/app-bundle-format?hl=zh_cn#multi_apks](https://developer.android.com/guide/app-bundle/app-bundle-format?hl=zh_cn#multi_apks)
[https://developer.android.com/studio/command-line/bundletool](https://developer.android.com/studio/command-line/bundletool)
[https://developer.android.com/guide/app-bundle/dynamic-delivery](https://developer.android.com/guide/app-bundle/dynamic-delivery)
[https://developer.android.com/guide/app-bundle/test](https://developer.android.com/guide/app-bundle/test)
[https://support.google.com/googleplay/android-developer/answer/9842756?hl=zh-Hans#](https://support.google.com/googleplay/android-developer/answer/9842756?hl=zh-Hans#)