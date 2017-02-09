## Firebase Notification配置

#### 前言：  
以前项目中用过GCM，现在Google收购了FireBase之后现在需要用到推送,看了下文档发现GCM并入了FCM,相关API和用法基本一致，趁此机会整理一下基本配置方法。

### 基本配置  
##### [官方文档](https://firebase.google.cn/docs/android/setup)
##### 运行要求：android2.3以上及google play service 9.6.1版本以上的设备  

##### 1. 在firebase中创建项目后创建android子项目填写相关信息 包括：package id  
##### 2. 复制生成的文件放到module目录下   
##### 3. 在project目录的build.gradle下添加依赖 

```
 dependencies {
 classpath 'com.google.gms:google-services:3.0.0' 
 }
``` 

##### 4. 在module目录的build.gradle下添加依赖 

```
 dependencies {
    compile 'com.google.firebase:firebase-messaging:9.6.1'
    compile 'com.google.android.gms:play-services:9.6.1'
}
//底部添加plugin   
apply plugin: 'com.google.gms.google-services'
```  

##### 5. 在BaseAvtivity中检查GoogleService是否可用：

    @Override
    protected void onResume() {
        super.onResume();
        isGooglePlayServicesAvailable();
        handleFcmDataExtras();
    }

    public void isGooglePlayServicesAvailable() {
        int state = GoogleApiAvailability.getInstance().isGooglePlayServicesAvailable(this);
        if (state != SUCCESS)
            GoogleApiAvailability.getInstance().getErrorDialog(this, state, 0).show();
    }

##### 6.  继承FirebaseInstanceIdservice注册token

    @Override
    public void onTokenRefresh() {
        // 获取 InstanceID
        String refreshedToken = FirebaseInstanceId.getInstance().getToken();
        Log.d(TAG, "Refreshed token: " + refreshedToken);

        sendRegistrationToServer(refreshedToken);
    }

    private void sendRegistrationToServer(String refreshedToken) {
        // 在此方法中将InstanceID发送给app的服务器，用于定向发送推送消息。
    }

##### 7.继承FirebaseMessagingService以及在Luncher Activity进行接收消息/数据的配置。

* FireBaseMessagingService中

```
   private final String TAG = "MyFirebaseMsgService";

@Override
    public void onMessageReceived(RemoteMessage remoteMessage) {

        //用于处理主题消息
        //并不是所有的消息都在这里处理，如果APP处在后台接收带数据的通知，那么数据会放在启动activity的Intent中
        if ((remoteMessage.getFrom().startsWith("/topics/"))) {
            String topic = remoteMessage.getFrom().replace("/topics/", "");
            Log.d(TAG, "From: " + remoteMessage.getFrom());
            Log.d(TAG, "From: " + topic);
        }
        // 检查是否包含数据
        if (remoteMessage.getData().size() > 0) {
            Log.d(TAG, "Message data payload: " + remoteMessage.getData());
        }

        // 检查是否包含通知
        if (remoteMessage.getNotification() != null) {
            Log.d(TAG, "Message Notification Body: " + remoteMessage.getNotification().getBody());
            sendNotification(remoteMessage.getNotification().getTitle(), remoteMessage.getNotification().getBody());
        }
        //如果有其它相关自定义操作，在这里完成
    }

    /**
     *  创建Notification发送给
     */
    private void sendNotification(String title, String messageBody) {
        Intent intent = new Intent(this, MainActivity.class);
        intent.addFlags(Intent.FLAG_ACTIVITY_CLEAR_TOP);
        PendingIntent pendingIntent = PendingIntent.getActivity(this, 0 /* Request code */, intent,
                PendingIntent.FLAG_ONE_SHOT);

        Uri defaultSoundUri = RingtoneManager.getDefaultUri(RingtoneManager.TYPE_NOTIFICATION);
        NotificationCompat.Builder notificationBuilder = new NotificationCompat.Builder(this)
                .setSmallIcon(R.drawable.ic_media_play)
                .setContentTitle("FCM Message")
                .setContentTitle(title)
                .setContentText(messageBody)
                .setAutoCancel(true)
                .setSound(defaultSoundUri)
                .setContentIntent(pendingIntent);

        NotificationManager notificationManager =
                (NotificationManager) getSystemService(Context.NOTIFICATION_SERVICE);

        notificationManager.notify(0 /* ID of notification */, notificationBuilder.build());
    }
```
* 在Launcher Activity中

```
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        // 当Notification的类型是message+data时，data在此处理
        if (getIntent().getExtras() != null) {
            for (String key : getIntent().getExtras().keySet()) {
                Object value = getIntent().getExtras().get(key);
                Log.d(TAG, "Key: " + key + " Value: " + value);
            }
        }}
```

##### [官方Sample](https://github.com/firebase/quickstart-android/tree/master/messaging) 
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
