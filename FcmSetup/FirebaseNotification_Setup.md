## Firebase Notification配置

#### 前言：  
#### 以前项目中用过GCM，最近Google收购了FCM之后现在需要用到FCM,读了下文档发现相关API和用法基本一致，趁此机会整理一下基本配置方法。

### 基本配置  
#### [官方文档](https://firebase.google.cn/docs/android/setup)
#### 运行要求：android2.3以上及google play service 9.6.1版本以上的设备  

#### 1. 在firebase中创建项目后创建android子项目填写相关信息 包括：package id  
#### 2. 复制生成的文件放到module目录下   
#### 3. 在project目录的build.gradle下添加依赖 

```
 dependencies {
 classpath 'com.google.gms:google-services:3.0.0' 
 }
``` 

#### 4. 在module目录的build.gradle下添加依赖 

```
 dependencies {
    compile 'com.google.firebase:firebase-messaging:9.6.1'
    compile 'com.google.android.gms:play-services:9.6.1'
}
```

#### 底部添加plugin   

```
apply plugin: 'com.google.gms.google-services'
```  

#### 5. 在项目中判断google play service是否可用、继承FirebaseInstanceIdservice、FirebaseMessagingService以及Luncher Activity进行接收消息的配置。（参考[官方Sample](https://github.com/firebase/quickstart-android/tree/master/messaging)）  
---
##### [官方文档](https://firebase.google.cn/docs/notifications/android/console-audience)
### 推送方式  
1. 根据app id推送
2. 根据设备的fcm Id推送 
3. 根据用户订阅的topic推送

### 消息类型
1. 通知
2. 数据
3. 通知+数据

### app接收消息时的行为
根据app在前台/后台、消息类型的不同分别作出不同的行为。 

| 应用状态        | 通知              | 数据  | 两者 |
| ------------- |:-----------------:|:-----:|:---:|
| 前台           | onMessageReceived | onMessageReceived | onMessageReceived |
| 后台           | 系统托盘           | onMessageReceived|通知：系统托盘;数据：Intent的extra中|
