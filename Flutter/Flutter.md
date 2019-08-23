# Flutter 介绍 & 经验总结

## 前言
Flutter 已经推出2年了，虽然一直在关注，但还是想等生态成熟一点再去踩坑。近期有一个需要使用跨平台技术的项目，在讨论后，我们选择使用 Flutter。开发完成之后，我这里总结一些重要的点，供大家参考。  
当然，要学习的话最后还是需要读一遍文档，然后自己 Coding。

## 环境配置：

参考[官方文档](https://flutter-io.cn/docs/get-started/install/macos)


## Dart 语言

Flutter 采用 Dart 语言，我使用之后的感受就是： 语法基本等于 Java + Javascript + 另外一些常见的语法，没太大学习成本，也没太大亮点，下面列一些值得一提的点。

* 所有变量都是对象
* 静态语言
* 支持闭包
* 方法是顶级的
* 支持反射（Flutter 不支持反射）
* 没有可见性修饰符 属性/类前加`_`就是 private
* Stream : 支持 map... 各类操作符，订阅等
* 异步：Dart 的异步操作也通过 `Futrue`（同 Javascript 中的 `Promise`） 的方式实现，也支持 `async` `await` 语法糖（自动包装为`Futrue`）。这并不是 Dart 特有的特性，网上有大量资料可以参考。
* 赋值操作符
  * ?:
  * ??
  * ??=
  
* 可选方法参数
 
```
 void setUser(String name,{id = '0'});
 //调用
 setUser('mario',id : '01');
```

* 联级操作符

```
   var profit = Profit()
     ..fund = 'fund'
     ..profit = 'profit'
     ..profitValue = 'profitValue';
```

* dynamic 可以指代任何类型，不会进行类型检查。

```
var a = 'test';
(a as dynamic).hello();//编译器不会报错
```



## Flutter

### Widget 概念

> 在Flutter中几乎所有的对象都是一个Widget。与原生开发中“控件”不同的是，Flutter中的Widget的概念更广泛，它不仅可以表示UI元素，也可以表示一些功能性的组件如：用于手势检测的 GestureDetector widget、用于APP主题数据传递的Theme等等，而原生开发中的控件通常只是指UI元素。

我的理解为 Widget 的工作 = HTML + CSS 的工作。而且很多配置样式的属性名字和 CSS 中的名字差不多。

Widget 分为 `StatelessWidget` `StatefulWidget` 两种，他们的核心方法都是通过`build()`方法返回一个 Widget 。

```
  @protected
  Widget build(BuildContext context);
```

* `StatelessWidget` 的`build()`在 Widget 中。  
* `StatefulWidget`由于必须创建相应的 `State<T extends Widget>` ,所以包括`build()`在内的相关生命周期方法都在`State`中。  
下面是`State`的生命周期，由于一个画面也是一个 Widget 所以也是一个画面的生命周期。

![widget_lifecyle.jpg](https://github.com/xiejinpeng007/XLearnNotes/blob/master/Flutter/widget_lifecyle.jpg?raw=true)

### Widget 目录 ( [link](https://flutterchina.club/widgets/material/) )

![widgets.png](https://github.com/xiejinpeng007/XLearnNotes/blob/master/Flutter/widgets.png?raw=true)

上面是官方提供的所有的 Widget，可以看到基本上所有UI相关的内容都是通过不同类型的 Widget 来实现，通过`child/children`参数进行嵌套。
#### 不同风格的 Widget
除了基础 Widget 外，官方提供了 Material(Android) + Cupertino(ioS) 两种视觉风格的 Widget。  
例如你可以在使用一个 Marterial 风格的`RaisedButton`或是 Cuptino 风格的`CupertinoButton`，再也不用担心设计师让 Android 照着 ioS 做成一样了。  
#### Layout Widget
还有用来控制布局的 Layout Widget ,作为容器来使用，看名字都大概知道什么作用了。

* Container
* Padding
* Center
* Stack
* Column
* Row
* Expanded
* ListView

#### 交互模型 Widget
控制点击、滑动等交互的 Widget。  
在 Flutter 里点击事件并不是`setOnClickListener`的方式 ，而是给 Widget 外层加一层交互 Widget ，如点击可使用`GestureDetector`。
例如给上面 Splash 画面中的`Image`加一个点击事件。

```
  @override
  Widget build(BuildContext context) {
    return Container(
      height: double.infinity,
      width: double.infinity,
      child: Image.asset('images/logo.png'),
    );
  }
  
  ==>
  
  @override
  Widget build(BuildContext context) {
    return Container(
      height: double.infinity,
      width: double.infinity,
      child: GestureDetector(
        onTap: () {
          //点击事件
        },
        child: Image.asset('images/logo.png'),
      ),
    );
  }
```

#### Sample

所以，一个最基本的 Widget 长什么样？这是一个带有是否 login 检查的 Splash 画面。

* StatelessWidget

```
class SplashPage extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    checkLogin();
    return Container(
      height: double.infinity,
      width: double.infinity,
      child: Image.asset('images/logo.png'),
    );
  }

// 使用 async 语法自动包装为 Futrue，也就是说这个方法是异步的。
  checkLogin(BuildContext context) async {
    var sp = await SharedPreferences.getInstance();
    var token = sp.getString("X-Auth-Token");
    if (token != null && token != "")
      Navigator.pushNamedAndRemoveUntil(
          context, HomePage.routeName, (_) => false);
    else
      Navigator.pushNamed(context, LoginRegisterPage.routeName);
  }
}

```

* StatefulWidget

```
class SplashPage extends StatefulWidget {
  //创建相应的 State
  @override
  State createState() => _SplashState();
}

class _SplashState extends State<SplashPage> {
  @override
  void initState() {
    super.initState();
    checkLogin(context);
  }

  @override
  Widget build(BuildContext context) {
    return Container(
      height: double.infinity,
      width: double.infinity,
      child: Image.asset('images/logo.png'),
    );
  }

// 使用 async 语法自动包装为 Futrue，也就是说这个方法是异步的。
  checkLogin(BuildContext context) async {
    var sp = await SharedPreferences.getInstance();
    var token = sp.getString("X-Auth-Token");
    if (token != null && token != "")
      Navigator.pushNamedAndRemoveUntil(
          context, HomePage.routeName, (_) => false);
    else
      Navigator.pushNamed(context, LoginRegisterPage.routeName);
  }
  
  @override
  void dispose() {
      super.dispose();
    }
}

```

### App 结构

![counterAppwidgertree.jpg](https://github.com/xiejinpeng007/XLearnNotes/blob/master/Flutter/counterAppwidgertree.jpg?raw=true)

上图是整个 Flutter App 的结构，从父节点开始分别是：

1. `MyApp`: 整个 App 的入口在`main.dart`的`main()`函数中，调用 `runApp(MyApp())`,而 MyApp 也是一个 Widget，只不过用来定义一些全局的内容，例如主题、多语言，路由
2. `MaterialApp`: 一个 Material 风格的主题，对应的还有 CupertinoApp。
3. `MyHomePage` `MyHomePageState` : 一个画面，也是 Widget。
4. `Scaffold` : 定义了一个画面的一些基本效果，比如这里 AppBar、滑动效果等采用 Material 风格，另外还有 ioS 风格的 `CupertinoPageScaffold`。
5. 剩下就是一些基本的组件。

一个基本的 main.dart 大概长这样：

```
void main() async {
  runApp(MyApp());
}

class MyApp extends StatelessWidget {
  static final navigatorKey = GlobalKey<NavigatorState>();
  static NavigatorState get navigator => navigatorKey.currentState;

  @override
  Widget build(BuildContext context) {
    return  CupertinoApp(
        title: '',
        theme: CupertinoThemeData(
          primaryColor: Color(0xFFFFFFFF),
          barBackgroundColor: Color(0xFF515669),
          scaffoldBackgroundColor: Color(0xFF3C3B45),
        ),
        navigatorKey: navigatorKey,
        routes: {
          HomePage.routeName: (_) => HomePage(),
          LoginRegisterPage.routeName: (_) => LoginRegisterPage(),
          LoginPage.routeName: (_) => LoginPage(),
          ForgetPswPage.routeName: (_) => ForgetPswPage(),
          RegisterPage.routeName: (_) => RegisterPage(),
        },
        ),
        home: SplashPage(),
    );
  }
}

```

  
* `theme` 定义了一个 ioS 风格的 CupertinoApp 主题（实际开发中可能需要同时使用 Material Cupertino 风格控件所以需要自定义主题）
* `routes` 参数注册路由表
* `home` 参数设置首次加载的 Splash 画面

### 路由

和 Web 中的路由类似，通过在路由表注册相应的 url 和画面。基本方法

* push / pushNamed / pushNamedAndRemoveUntil/...
* pop / popUntil / ...

基本使用：

```
// pushNamed 的定义
Future pushNamed(BuildContext context, String routeName,{Object arguments})

//打开一个画面，传一个00
Navigator.of(context).pushNamed("home_page", arguments: '00');

//新画面接受参数
var arg = ModalRoute.of(context).settings.arguments);

//关闭一个画面，返回一个01
Navigator.of(context).pop(01);

```

* 实际上更好的方法来处理传值的问题
* 可以看到`pushNamed`方法返回值是一个`Future` ，说明是一个异步操作，因为可以接受打开的画面`pop`关闭时返回的`result` ，此处在`pop`时返回了一个 01，那么就可以这样接收到。

```
var result = async Navigator.of(context).pushNamed("home_page", arguments: 'arg');
```

### 网络请求和序列化

Flutter 的 网络请求库没有特别完美的，目前使用的是 [Dio](https://github.com/flutterchina/dio) ,大致是一个简化版的 okhttp 。

由于 Flutter 禁止使用反射，因为运行时反射会干扰 Dart 的 tree shaking，所以类似 Gson 这样通过反射进行序列化的方式就行不通了。  
目前大概的解决方案有两种：

* 手写：`Dio` 会把返回值解析为 Map/List ，所以可以这样手写:

```
  Future<Profits> requestProfits() async {
    var response = await dio.get("u/profits");
    var data = response.data;
    print("requestProfits:$data");

    var profit = Profit()
      ..fund = data['profit']["fund"]
      ..profit = data['profit']["profit"]
      ..profitValue = toMoney(data['profit']['profitValue']);

    return Profits()
      ..miningProfit = data['miningProfit']
      ..lastMiningProfit = data['lastMiningProfit']
      ..shareProfit = data['shareProfit']
      ..lastShareProfit = data['lastShareProfit']
      ..tradeProfit = data['tradeProfit']
      ..lastTradeProfit = data['lastTradeProfit']
      ..vipProfit = data['vipProfit']
      ..lastVipProfit = data['lastVipProfit']
      ..profit = profit;
  }
```

* 生成代码：使用 [json_serializable](https://pub.dev/packages/json_serializable)

```
//user.dart

import 'package:json_annotation/json_annotation.dart';

// user.g.dart 将在我们运行生成命令后自动生成
part 'user.g.dart';

///这个标注是告诉生成器，这个类是需要生成Model类的
@JsonSerializable()

class User{
  User(this.name, this.email);

  String name;
  String email;
  //不同的类使用不同的mixin即可
  factory User.fromJson(Map<String, dynamic> json) => _$UserFromJson(json);
  Map<String, dynamic> toJson() => _$UserToJson(this);  
}
```
当然，还是需要写 `fromJson` `toJson` 的模板代码，也可以通过生成的方式解决。

### 平台特定代码

Flutter 主要是负责了UI部分的构建，各平台特定的代码还是要通过原生实现，主要用两种方法处理：

* `Platform Channel` : 大概就是 Flutter 端和原生端注册约定好 `platform_channel_name` 的 `Platform Channel`  ，然后调用方法和传参，另一端解析就行了。具有原生能力的 plugin 也就是这样实现的。比如

```
//flutter
MethodChannel('method_channel_mobile').invokeMethod('sendMobile','13000000000')

//Android MainActivity

MethodChannel(flutterView, MOBILE_CHANNEL)
            .setMethodCallHandler { methodCall, result ->
                when {
                    TextUtils.equals(methodCall.method, "mobile") -> {
                        mobile = methodCall.arguments.toString()
                        result.success("success")
                    }
                     result.notImplemented()
                }
            }
```

* `PlatformView` 直接嵌套原生的 View 到 Flutter 中，但这样做效率不高。另外需要注意的是不要传入一个 view 到`PlatformView`中，否则可能出现 Flutter 端多次调用该`PlatformView`的时候状态会共存，以及不会销毁。

```
// 定义一个用于的 PlatformView 和 PlatformViewFactory 用于实例化 Native View 
class ButtonFactory(
    private val context: Context
) : PlatformViewFactory(StandardMessageCodec.INSTANCE) {

    override fun create(p0: Context?, p1: Int, p2: Any?): PlatformView {
        return ButtonPlatformView(context)
    }
    class ButtonPlatformView(
        private val context: Context
    ) : PlatformView {

        override fun getView(): Button {
            return Button(context)
        }
        override fun dispose() {
        }
    }
}

//在 MainActivity 中注册
        registrarFor("native_view").platformViewRegistry()
            .registerViewFactory("native_view",ButtonPlatformFactory)
```

### Widget 嵌套的问题

网上对 Flutter 嵌套讨论的比较多的问题就是，UI 复杂了以后，嵌套层数太多。
确实有这个问题，之前说了 Widget 不光是 View 还包括配置文件，所以一个类似 Button 这样的 Widget 可能就需要嵌套3 4层。  
下面是我写的一个登录画面的登录按钮，感受一下:

```
  CupertinoButton _loginButton() {
    return CupertinoButton(
      padding: EdgeInsets.all(0),
      child: Container(
          width: double.infinity,
          height: 45,
          decoration: BoxDecoration(
            gradient: LinearGradient(
                begin: Alignment.topCenter,
                end: Alignment.bottomCenter,
                colors: _isLoginAvailable
                    ? <Color>[Color(0xFF657FF8), Color(0xFF4260E8)]
                    : <Color>[Color(0xFFCBCFE2), Color(0xFF73788F)]),
            borderRadius: BorderRadius.all(Radius.circular(6)),
          ),
          child: Center(
            child: Text(
              "登 录",
              textAlign: TextAlign.center,
              style: TextStyle(
                fontSize: 18,
                color: _isLoginAvailable ? Colors.white : Color(0x76FFFFFF),
                fontWeight: FontWeight.bold,
              ),
            ),
          )),
//      shape: RoundedRectangleBorder(borderRadius: BorderRadius.circular(6)),
      onPressed: !_isLoginAvailable ? null : _startCustomFlow,
    );
  }
```

实际上这还是只是结构+样式部分，不包括点击后的逻辑。  
甚至你可以看到 Button 中的文字也是通过嵌套一个 Widget 来实现的，但这也是 Flutter 的一个优势，不再需要写自定义 Widget 的人去提供大量像文字能不能加粗，变色、斜体等等细节的样式，直接让你传一个 Widget 自行处理，类似的情况还有很多。  
另外一个问题是 Widget State 的状态可能太多，包括各个 Widget 的状态和画面的状态堆在一起，想起了当年原生 Android 一个 Activity 50个变量的恐惧。  
但我认为这些主要还是因为 Flutter 处于发展的初期，还没有太成熟的架构，目前官方提供了状态管理的库 `Provider`。
我目前的解决方案是尽量提成方法和独立的Widget：

* 对于有整个页面无关局部状态的 Widget 提成一个独立的 `StatefulWidget`。
* 对于没有局部状态的，需要重用就提成一个`StatelessWidget`，不需要就抽成一个方法，返回 Widget，参考上面的 Button 。
* 最后在`build()`方法中只描述整个画面的结构。

例如一个 login 画面的`build()`方法我是这样写的：

```
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      home: Scaffold(
        appBar: cusAppBar(context, elevation: 0),
        backgroundColor: $3C3B45,
        body: Stack(
          children: <Widget>[
            SingleChildScrollView(
              child: Container(
                margin: EdgeInsets.only(left: 15, right: 15),
                child: Column(
                  children: <Widget>[
                    _logo(),
                    Form(
                      onChanged: _onFormChanged,
                      child: Column(
                        children: <Widget>[
                          _phoneRow(),
                          _divider(),
                          _passwordColumn(),
                          _forgetPswText(),
                          _loginButton(),
                          _registerText()
                        ],
                      ),
                    ),
                  ],
                ),
              ),
            ),
          ],
        ),
      ),
    );
  }
```

### 热重载（HotReload）

Flutter 的热重载是广受欢迎的一个特性，重要原因则是 Debug 模式采用 JIT 编译，release 模式采用 AOT 编译。实际用下来效果不错。

### 问题

* 编译偶尔遇到的一个问题：  
问题：Waiting for another flutter command to release the startup lock...   
解决：rm ./flutter/bin/cache/lockfile



##### 最后，以上只是总结一些重要的点，最终官方文档肯定是要读一遍的,熟悉大部分 Widget 的用法: [文档](https://flutterchina.club/docs/)

### 总结

优点：

* `Android iOs 两端 UI 高度一致`：由于 Flutter 使用自己的一套绘制 UI 的引擎和逻辑，完全不使用 Native View，仅仅调用原生的绘制接口，所以几乎可以做到两个平台的 UI 一模一样，这也是 Flutter 还要做 Web maCos 等全平台的原因。我在开发期间一直使用 Android 进行调试，最后在 Ios 上跑的时候，几乎没有什么差别(虽然目前 UI 也不太复杂)。
* `接入原生相对容易`：需要原生实现的功能通过`PlatformChannel`和 `PlatformView`也大多都能实现，还可以通过`PlatformChannel`来启动一个原生的`Activity/Fragment`实现。（比如扫一扫功能）
* `贵族血统`：Google 的全力支持，国内大厂也都在积极尝试。
* `初步可用的程度`：目前已经完成了一个小项目的开发，在和原生交互不多的情况下还没有遇到太大的坑。

缺点：

* `基础功能的缺失`：很多基础的功能也需要用 plugin 通过原生来实现，比如 Webview Map 这些组件，更不要说一些 SDK ,几乎都需要自己写 plugin。
* `跨平台的通信`：对于大量使用`MethodChannel`进行通信以及各平台间API有差异的情况下，设计和维护的问题。
* `性能`：目前原生 Flutter 在帧数上接近原生，用户使用体验接近，但内存开销更大，尤其在视频方面。

总的来说：我的看法是，比较看好 Flutter 跨更多平台的前途，目前来说适合用来开发和原生平台 API 交互不那么复杂的 App 。
