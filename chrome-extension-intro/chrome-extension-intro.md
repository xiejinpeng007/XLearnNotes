# Chrome Extensions 开发简介

### 前言

最近想写一个对网页进行自动填充、跳转等操作的一个脚本。联想到平时用了那么多插件，调查了一下之后发现 Chrome 插件可以做很多事情，翻译类、广告拦截类、修改页面样式类（主题、暗黑模式）、效率类（快捷键、鼠标手势）、以及一系列小工具。

### Chrome 插件是什么？

Chrome 插件相信各位用的很多很熟悉了，根据名字来看，其实`插件/拓展`这个描述挺准的，Chrome Extensions 能在原本网页功能的基础上通过操作 DOM 来添加/修改 JS 、CSS 、HTML 达到功能的增加、样式的修改等目的。

### 开发 Chrome Extensions 需要具备的知识

* JavaScript HTML CSS 

## 核心部分
### manifest.json

`manifes.json`是 Chrome 插件最重要也是唯一必须的文件，用来配置所有和插件相关的配置，必须放在根目录。其中，`manifest_version`、`name`、`version` 3个是必不可少的，description 和 icons 是推荐的。
这里是以 V2 版本为例，最新的 manifest 版本刚刚被 Google 升到了 V3。

```
{
	// manifest 版本，最新的是 3
	"manifest_version": 2,
	// 插件的名称
	"name": "demo",
	// 插件的版本
	"version": "1.0.0",
	// 插件描述
	"description": "简单的Chrome扩展demo",
	// 图标
	"icons":
	{
		"16": "img/icon.png",
		"48": "img/icon.png",
		"128": "img/icon.png"
	},
	// 会一直常驻的后台JS或后台页面
	"background":
	{
		// 2种指定方式，如果指定JS，那么会自动生成一个背景页
		"page": "background.html"
		//"scripts": ["js/background.js"]
	},
	// 浏览器右上角图标设置，browser_action、page_action、app必须三选一
	"browser_action": 
	{
		"default_icon": "img/icon.png",
		// 图标悬停时的标题，可选
		"default_title": "这是一个示例Chrome插件",
		"default_popup": "popup.html"
	},
	// 当某些特定页面打开才显示的图标
	/*"page_action":
	{
		"default_icon": "img/icon.png",
		"default_title": "我是pageAction",
		"default_popup": "popup.html"
	},*/
	// 需要直接注入页面的JS
	"content_scripts": 
	[
		{
			//"matches": ["http://*/*", "https://*/*"],
			// "<all_urls>" 表示匹配所有地址
			"matches": ["<all_urls>"],
			// 多个JS按顺序注入
			"js": ["js/jquery-1.8.3.js", "js/content-script.js"],
			// JS的注入可以随便一点，但是CSS的注意就要千万小心了，因为一不小心就可能影响全局样式
			"css": ["css/custom.css"],
			// 代码注入的时间，可选值： "document_start", "document_end", or "document_idle"，最后一个表示页面空闲时，默认document_idle
			"run_at": "document_start"
		},
		// 这里仅仅是为了演示content-script可以配置多个规则
		{
			"matches": ["*://*/*.png", "*://*/*.jpg", "*://*/*.gif", "*://*/*.bmp"],
			"js": ["js/show-image-content-size.js"]
		}
	],
	// 权限申请
	"permissions":
	[
		"contextMenus", // 右键菜单
		"tabs", // 标签
		"notifications", // 通知
		"webRequest", // web请求
		"webRequestBlocking",
		"storage", // 插件本地存储
		"http://*/*", // 可以通过executeScript或者insertCSS访问的网站
		"https://*/*" // 可以通过executeScript或者insertCSS访问的网站
	],
	// 普通页面能够直接访问的插件资源列表，如果不设置是无法直接访问的
	"web_accessible_resources": ["js/inject.js"],
	// 插件主页，这个很重要，不要浪费了这个免费广告位
	"homepage_url": "https://www.baidu.com",
	// 覆盖浏览器默认页面
	"chrome_url_overrides":
	{
		// 覆盖浏览器默认的新标签页
		"newtab": "newtab.html"
	},
	// Chrome40以后的插件配置页写法，如果2个都写，新版Chrome只认后面这一个
	"options_ui":
	{
		"page": "options.html",
		// 添加一些默认的样式，推荐使用
		"chrome_style": true
	},
	// 向地址栏注册一个关键字以提供搜索建议，只能设置一个关键字
	"omnibox": { "keyword" : "go" },
	// 默认语言
	"default_locale": "zh_CN",
	// devtools页面入口，注意只能指向一个HTML文件，不能是JS文件
	"devtools_page": "devtools.html"
}
```

#### Sample

匹配不同的 url 执行不同的脚本

```
{
    "name": "自动预约",
    "version": "1.0",
    "description": "sede.administracionespublicas",
    "permissions": ["tabs","activeTab","declarativeContent","storage"],
    "background": {
      "scripts": ["background.js"],
      "persistent": false
    },
    "browser_action": {
        "default_popup": "popup.html"
      },
      "content_scripts": [
        {
          "matches": ["https://sede.administracionespublicas/icpplustiem/index.html"],
          "run_at": "document_end",
          "js": ["autofill_index.js"]
        },{
          "matches": ["https://sede.administracionespublicas/icpplustiem/citar*"],
          "run_at": "document_end",
          "js": ["autofill_citar.js"]
        },
        {
          "matches": ["https://sede.administracionespublicas/icpplustiem/acInfo"],
          "run_at": "document_end",
          "js": ["autofill_acinfo.js"]
        },
        {
          "matches": ["https://sede.administracionespublicas./icpplustiem/acEntrada"],
          "run_at": "document_end",
          "js": ["autofill_acentrada.js"]
        },
        {
          "matches": ["https://sede.administracionespublicas/icpplustiem/acValidarEntrada"],
          "run_at": "document_end",
          "js": ["autofill_acvalidar.js"]
        },
        {
          "matches": ["https://sede.administracionespublicas.gob.es/icpplustiem/acCitar"],
          "run_at": "document_end",
          "js": ["autofill_acCitar.js"]
        },
        {
          "matches": ["https://sede.administracionespublicas.gob.es/icpplustiem/acVerFormulario"],
          "run_at": "document_end",
          "js": ["autofill_acVerFormulario.js"]
        }
      ],
    "manifest_version": 2
  }

```


![](https://developer-chrome-com.imgix.net/image/BrQidfK9jaQyIHwdw91aVpkPiib2/CNDAVsTnJeSskIXVnSQV.png?auto=format&w=400)

![和其它组件通信](https://developer-chrome-com.imgix.net/image/BrQidfK9jaQyIHwdw91aVpkPiib2/466ftDp0EXB4E1XeaGh0.png?auto=format&w=400)

### Content Scripts

>
Content scripts are files that run in the context of web pages. By using the standard Document Object Model (DOM), they are able to read details of the web pages the browser visits, make changes to them and pass information to their parent extension.

[`Content Scripts`](https://developer.chrome.com/docs/extensions/mv2/content_scripts/) 是实现插件的拓展能力非常重要的一部分， Contents Scripts 让我们通过操作 DOM 来实现对页面动态/静态地注入 JS CSS。   
Contents Scripts 和原生页面共享 DOM 但是不共享 JS（无法修改原始 JS），使用的 API 也比较有限，但可以通过和权限更高的`background scripts`通讯来实现。


```
{
	// 需要直接注入页面的JS
	"content_scripts": 
	[
		{
			//"matches": ["http://*/*", "https://*/*"],
			// "<all_urls>" 表示匹配所有地址
			"matches": ["<all_urls>"],
			// 多个JS按顺序注入
			"js": ["js/jquery-1.8.3.js", "js/content-script.js"],
			// 注入 css
			"css": ["css/custom.css"],
			// 代码注入的时间，可选值： "document_start", "document_end", or "document_idle"，默认document_idle
			"run_at": "document_start"
		}
	],
}
```

Content Scripts 可以直接使用的 API：

* i18n
* storage
* runtime:
	* connect
	* getManifest
 	* getURL
	* id
	* onConnect
	* onMessage
	* sendMessage

## background scripts

[`background scripts`](https://developer.chrome.com/docs/extensions/mv2/background_pages/) 是插件的事件处理中心，会在有任务的时候尽可能长时间地运行在后台，用来监听一些重要的事件（插件被安装、导航页被打开、Tab 被打开、Tab被跳转以及 contents scripts 传过来的事件等等），还可以进行一些类似 Tab 创建/跳转的操作，权限也较大。


```
{
  "name": "Awesome Test Extension",
  ...
  "background": {
    "scripts": ["background.js"],
    "persistent": false
  },
  ...
}
```

注册 Listener Sample

```
//插件安装了后注册一个监听右键选中文本的事件 (比如选中本文后右键可以通过 Google 搜索 “xxx”)
chrome.runtime.onInstalled.addListener(function() {
  chrome.contextMenus.create({
    "id": "sampleContextMenu",
    "title": "Sample Context Menu",
    "contexts": ["selection"]
  });
});

// This will run when a bookmark is created.
chrome.bookmarks.onCreated.addListener(function() {
  // do something
});

```

### Communication between pages  & Storage & Message passing

插件间不同组件通信可以通过几个方法来实现

*  ` chrome.extension`  API 下的 getViews() getBackgroundPage()...
*  `storage` API  HTML5 `web storage API`
*  `message passing`

#### Storage API
关于 [Storage API](https://developer.chrome.com/docs/apps/app_storage/) 有很多内容，这里主要介绍常用的的 [LocalStorage API](https://developer.chrome.com/docs/extensions/reference/storage/)

使用 `chrome.storage.sync.* `存储的数据会同步到其它登录过的设备

```
chrome.storage.sync.set({key: value}, function() {
  console.log('Value is set to ' + value);
});

chrome.storage.sync.get(['key'], function(result) {
  console.log('Value currently is ' + result.key);
});
```

使用` chrome.storage.local.* `  存储的数据只会存储在本地
 
```
chrome.storage.local.set({key: value}, function() {
  console.log('Value is set to ' + value);
});

chrome.storage.local.get(['key'], function(result) {
  console.log('Value currently is ' + result.key);
});

监听数据的变化
```

```
chrome.storage.onChanged.addListener(function(changes, namespace) {
  for (var key in changes) {
    var storageChange = changes[key];
    console.log('Storage key "%s" in namespace "%s" changed. ' +
                'Old value was "%s", new value is "%s".',
                key,
                namespace,
                storageChange.oldValue,
                storageChange.newValue);
  }
});
```

### Message passing

由于 contents scripts 的当前的 web page 而不是整个插件，所以通过 web page context 所能获取的信息/权限有限。这时需要使用各类消息机制来传递消息。

[Message passing](https://developer.chrome.com/docs/extensions/mv3/messaging/)有多种方式：

* Simple one-time requests 
* Long-lived connections
* Cross-extension messaging
* Native messaging

这里主要介绍常用的 `Simple one-time requests` 

#### Simple one-time requests 简单的一次性消息
简单的插件内部消息可以通过  `runtime.sendMessage` or `tabs.sendMessage` 发送一次性 JSON-serializable 消息来实现。

从`content scripts` 发送到 extension (如`background.js`)

```
chrome.runtime.sendMessage({greeting: "hello"}, function(response) {
  console.log(response.farewell);
});
```

从 extension（比如 `background.js`）发送到 `content scripts`

```
//也可以传 null 不指定 tabId
chrome.tabs.query({active: true, currentWindow: true}, function(tabs) {
  chrome.tabs.sendMessage(tabs[0].id, {greeting: "hello"}, function(response) {
    console.log(response.farewell);
  });
});
```

接收消息，此外如果多个页面都监听了消息，只有第一个调用 `sendResponse`的会成功发送。

```
chrome.runtime.onMessage.addListener(
  function(request, sender, sendResponse) {
    console.log(sender.tab ?
                "from a content script:" + sender.tab.url :
                "from the extension");
    if (request.greeting == "hello")
      sendResponse({farewell: "goodbye"});
  }
);
```

### Permission & API

类似 app ,使用 chrome API 的时候需要在 manifest.json 申明相关权限。

##### 比较常用用的一些API系列：

* chrome.tabs
* chrome.runtime
* chrome.webRequest
* chrome.window
* chrome.storage
* chrome.contextMenus
* chrome.devtools
* chrome.extension

##### 权限列表参考

* [Permission](https://developer.chrome.com/docs/extensions/mv3/declare_permissions/)
* [API](https://developer.chrome.com/docs/extensions/reference/)

## UI 交互部分
[Design the user interface](https://developer.chrome.com/docs/extensions/mv3/user_interface/)

### Popup

![](https://developer-chrome-com.imgix.net/image/BrQidfK9jaQyIHwdw91aVpkPiib2/JVduBMXnyUorfNjFZmue.png?auto=format&w=300)

是一个点击右上角工具栏图标后弹出一个页面，（通过[`chrome.browserAction`](https://developer.chrome.com/docs/extensions/reference/browserAction/) 配置）也是 chrome.browserAction 重要的一个配置。可以用来做成提供给用户对插件进行配置的窗口。

##### 通过 browser action or page action 来注册 popup
```
{
  "name": "Drink Water Event",
  ...
  "browser_action": {
    "default_popup": "popup.html"
  }
  ...
}
```

也可以通过 `browserAction.setPopup` or `pageAction.setPopup` 动态设置

```
chrome.storage.local.get('signed_in', function(data) {
  if (data.signed_in) {
    chrome.browserAction.setPopup({popup: 'popup.html'});
  } else {
    chrome.browserAction.setPopup({popup: 'popup_sign_in.html'});
  }
});
```

### Notification

全局弹窗使用可以 Chrome Notification 或 HTML5 Notification  
当前页面可以用原生 `alert('msg')`

```
chrome.notifications.create(null, {
	type: 'basic',
	iconUrl: 'img/icon.png',
	title: 'title',
	message: 'msg'
});
```

### Context menu

上下文菜单：显示在选中文字后的右键的菜单

![](https://developer-chrome-com.imgix.net/image/BrQidfK9jaQyIHwdw91aVpkPiib2/LhrliaEhN82maJmeNp7f.png?auto=format&w=400)

需要在 background.js 下创建，其中 title 中如果放的是数组的话，那么会显示二级菜单。

```
chrome.runtime.onInstalled.addListener(function() {
  for (let key of Object.keys(kLocales)) {
    chrome.contextMenus.create({
      id: key,
      title: kLocales[key],
      type: 'normal',
      contexts: ['selection'],
    });
  }
});

const kLocales = {
  'com.au': 'Australia',
  'com.br': 'Brazil',
  'ca': 'Canada',
  'cn': 'China',
  'fr': 'France',
  'it': 'Italy',
  'co.in': 'India',
  'co.jp': 'Japan',
  'com.ms': 'Mexico',
  'ru': 'Russia',
  'co.za': 'South Africa',
  'co.uk': 'United Kingdom'
};
```

### Commands

用来注册快捷键和相关的命令

```
{
  "name": "Tab Flipper",
  ...
  "commands": {
    "flip-tabs-forward": {
      "suggested_key": {
        "default": "Ctrl+Shift+Right",
        "mac": "Command+Shift+Right"
      },
      "description": "Flip tabs forward"
    },
    "flip-tabs-backwards": {
      "suggested_key": {
        "default": "Ctrl+Shift+Left",
        "mac": "Command+Shift+Left"
      },
      "description": "Flip tabs backwards"
    }
  }
  ...
}
```

### Override pages

用来覆盖 新Tab、历史记录、标签页这三种页面的任意一种。

```
{
  "name": "Awesome Override Extension",
  ...

  "chrome_url_overrides" : {
    "newtab": "override_page.html" //or history or bookmarks
  },
  ...
}
```

* [ API ](https://developer.chrome.com/docs/extensions/reference/)
* [ Samples ](https://github.com/GoogleChrome/chrome-extensions-samples)

### 打包 & 发布
插件可以在扩展程序页面（chrome://extensions/）选择`打包扩展程序`打包为 .crx 文件进行发布（需要注册成开发者），一般自用的话打开开发者模式选择`加载已解压的扩展程序`选择文件夹即可。

### Manifest V3

刚推出的新版本 [Manifest V3](https://developer.chrome.com/docs/extensions/mv3/intro/) 主要是一些 API 的变更，v2也可以继续使用。

### 总结

对于 Chrome Extensions 开发我也还在学习中，以上主要总结了常用且重要的配置、特性和 API ，还有很多其它 API 可以参考文档，
官方文档没有中文，但也写的很简洁易懂，官方 sample 更是详尽，几乎所有的 API 都有相应的 sample，建议参考。

* [chrome-extensions-mv3](https://developer.chrome.com/docs/extensions/mv3/)
* [chrome-extensions-samples](https://github.com/GoogleChrome/chrome-extensions-samples)

