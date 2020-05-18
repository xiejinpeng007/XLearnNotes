# WebRTC 介绍和网络视频通话的实现(Android+服务器端)

近期由于疫情原因，国内外线上会议使用率攀升，很多公司都推出了相关服务， Google 也把本来付费会议服务 Meeting 变为免费。  
之前调查过一个摄像头监控功能的 App ，用到了 WebRTC 相关的技术，做了技术调查之后发现 WebRTC 还可以做更多的事，于是做成了一个可以视频通话的客户端和服务器端，并做了一些技术的总结，
并且实际上现在大部分的网络视频(单人/多人)很多都基于相关技术进行拓展。

### 简介：

> WebRTC（Web Real-Time Communication）是一个支持网页浏览器进行实时语音对话或视频对话的API。它于2011年6月1日开源并在Google、Mozilla、Opera支持下被纳入万维网联盟的W3C推荐标准。

虽然叫 WebRTC ，实际上目前主流浏览器和操作系统都已经支持了相关API。它通过点对点（Point-to-Point）的方式进行通话，但也需要信令服务器来进行相关配置信息的交换。

***注：下面的原理介绍和 demo 实现，我都按照最标准易懂的流程来设计，实际的业务开发会根据情况进行调整。***

### 原理：

#### 基本的图示：
![](/Users/jinpengxie/Downloads/simple_arch.png)

从上图可以看AB互相呼叫的相关流程需要通过信令服务器中转，而视频/音频等流量数据是点对点直接传输的。

#### 重要 API 和相关协议：
* Network Stream API
 * MediaStream：MediaStream 用来表示一个媒体数据流。
 * MediaStreamTrack 在浏览器中表示一个媒体源。
* RTCPeerConnection
 * RTCPeerConnection：一个RTCPeerConnection对象允许用户在两个终端之间直接通讯。
 * RTCIceCandidate：表示一个ICE协议的候选者。
* RTCIceServer：表示一个ICE Server。
 * Peer-to-peer Data API
 * DataChannel：数据通道（DataChannel）接口表示一个在两个节点之间的双向的数据通道。
* [Session Description Protocol](https://developer.mozilla.org/en-US/docs/Web/API/WebRTC_API/Protocols#SDP) :一种用于描述在设备之间共享媒体的连接的数据格式.
*  [Interactive Connectivity Establishment (ICE) ](https://developer.mozilla.org/en-US/docs/Web/API/WebRTC_API/Protocols#ICE): 一个用于网络穿透的框架，其中使用 TURN/STUN 服务来实现。
*  [Session Traversal Utilities for NAT (STUN)](https://developer.mozilla.org/en-US/docs/Web/API/WebRTC_API/Protocols#STUN): 用于获取公网地址的协议
<div align="center"><img src="/Users/jinpengxie/Downloads/webrtc-stun.png" height="300"></div>
*  [Traversal Using Relays around NAT (TURN) ](https://developer.mozilla.org/en-US/docs/Web/API/WebRTC_API/Protocols#TURN): 用于中继数据的协议
<div align="center"><img src="/Users/jinpengxie/Downloads/webrtc-turn.png" height="300"></div>

#### 建立连接的基本流程：

1. 两端进行相关初始化（Socket 、ICE、流媒体等的配置）
2. A： 发起呼叫:创建用于连接的 `PeerConnection(PC)` 和自己的配置文件 `SessionDescription(SDP) `
   将 SDP 设置为 LocalDescription ,然后通过信令服务器将 SDP 转发给 B ,这个流程称之为 Offer 。
3. B 收到 SDP 后设置为 RemoteDescription ,创建自己的 SDP ，设置为 LocalDescription,然后通过信令服务器将 SDP 转发 给 A,这个过程称之为 Answer。
4. A 收到 SDP 后设置为 RemoteDescription
5. 在初始化时会进行 ICE 服务的配置，所以 ICE 服务成功后有回调，A B 在回调后将 ICE 的配置发送给对方，收到后分别设置到 ICE 配置中，则会进行最终的连接。
6. 连接成功后若已设置 DataChannel MediaStream ...等配置，那么相关回调会执行，此时即可获取数据。

期间，因为需要点对点的通信，而在公网上由于 NAT/firewalls 的限制，无法直接进行通讯，所以需要使用 ICE 框架来进行，ICE 框架内部使用 STUN / TURN 协议来实现。

* STUN： 上面已经解释了是用于获取公网IP的服务，Google 也提供了公共的服务器 `stun:stun.l.google.com:19302`
* TURN： 主要是用于客户端即使知道了互相的 IP ，由于 Symmetric NAT 的限制无法直接建立连接时用于转发媒体流数据的服务，这个一般来说需要自己搭建。

### Android 客户端的实现

客户端的功能包括了自定义服务器地址连接服务器、查看在线设备、选择设备进行视频通话

依赖库中 Webrtc 使用 Google 官方提供的， 服务器端和 Android 端使用了同样的 Socket 库，若服务器端没有什么限制推荐使用 OkHttp 自带的 Socket 通信功能。

```
    implementation 'org.webrtc:google-webrtc:1.0.28513'
    implementation 'com.github.nkzawa:socket.io-client:0.4.2'

```

#### WebRtcClient 的初始化

```
    init {
        //初始化 PeerConnectionFactory 配置
        PeerConnectionFactory.initialize(
            PeerConnectionFactory
                .InitializationOptions
                .builder(app)
                .createInitializationOptions()
        )
        
        //初始化视频编码/解码信息
        factory = PeerConnectionFactory.builder()
            .setVideoDecoderFactory(
                DefaultVideoDecoderFactory(eglContext)
            )
            .setVideoEncoderFactory(
                DefaultVideoEncoderFactory(
                    eglContext, true, true
                )
            )
            .createPeerConnectionFactory()

        // 初始化 Socket 通信
        val messageHandler = MessageHandler()

        try {
            socket = IO.socket(url)
        } catch (e: URISyntaxException) {
            e.printStackTrace()
        }

        socket?.on("id", messageHandler.onId)
        socket?.on("message", messageHandler.onMessage)
        socket?.on("ids", messageHandler.onIdsChanged)
        socket?.connect()

        //初始化 ICE 服务器创建 PC 时使用
        iceServers.add(PeerConnection.IceServer("stun:23.21.150.121"))
        iceServers.add(PeerConnection.IceServer("stun:stun.l.google.com:19302"))

        //初始化本地的 MediaConstraints 创建 PC 时使用，是流媒体的配置信息
        pcConstraints.mandatory.add(MediaConstraints.KeyValuePair("OfferToReceiveAudio", "true"))
        pcConstraints.mandatory.add(MediaConstraints.KeyValuePair("OfferToReceiveVideo", "true"))
        pcConstraints.optional.add(MediaConstraints.KeyValuePair("DtlsSrtpKeyAgreement", "true"))
    }
```

#### 开始建立连接

上面介绍的`建立连接的基本流程`提到了 A 呼叫 B 的话是由 A 开启 Offer 流程，由于我希望创建 PC 的时候知道自己的 id 和获取所有在线客户端，所以修改了一些流程，增加了`init` `readyToStream`的动作。

* 初始化 Socket 连接上服务器后会返回相应 clientId, 此时进行本地摄像头的初始化和本地媒体流的初始化最后进行向服务器发送准备初始化成功的指令。

```
    private fun getVideoCapturer() =
        Camera2Enumerator(app).run {
            deviceNames.find {
                isFrontFacing(it)
            }?.let {
                createCapturer(it, null)
            } ?: throw IllegalStateException()
        }

    fun startLocalCamera(name: String, context: Context) {
        //init local media stream
        val localVideoSource = factory.createVideoSource(false)
        val surfaceTextureHelper =
            SurfaceTextureHelper.create(
                Thread.currentThread().name, eglContext
            )
        (vc as VideoCapturer).initialize(
            surfaceTextureHelper,
            context,
            localVideoSource.capturerObserver
        )
        vc.startCapture(320, 240, 60)
        localMS = factory.createLocalMediaStream("LOCALMEDIASTREAM")
        localMS?.addTrack(factory.createVideoTrack("LOCALMEDIASTREAM", localVideoSource))
        webrtcListener.onLocalStream(localMS!!)

        try {
            val message = JSONObject()
            message.put("name", name)
            socket?.emit("readyToStream", message)
        } catch (e: JSONException) {
            e.printStackTrace()
        }
    }
```

* 此时已连上服务器并配置完毕，调用 `refreshIds` 获取已连接上服务器客户端，选择 id 进行呼叫 

```
  //发送消息的方法
private fun sendMessage(to: String, type: String, payload: JSONObject) {
        val message = JSONObject()
        message.put("to", to)
        message.put("type", type)
        message.put("payload", payload)
        socket?.emit("message", message)
    }
    
    fun refreshIds() {
        socket?.emit("refreshids", null)
    }
    
    fun callByClientId(clientId: String) {
        sendMessage(clientId, "init", JSONObject())
    }

```

* `readyToStream` `refreshIds`是为了实现查看在线设备相关功能，并非 WebRTC 的标准，下面的`建立连接的基本流程`是必要的流程。
接收消息后根据消息进入不同的响应流程以及具体的实现。

```
    private inner class MessageHandler {
		//建立 PC 交换 SDP ICE 等配置的事件
        val onMessage = Emitter.Listener { args ->
            val data = args[0] as JSONObject
            try {
                val from = data.getString("from")
                val type = data.getString("type")
                var payload: JSONObject? = null
                if (type != "init") {
                    payload = data.getJSONObject("payload")
                }
                //用于检查是否 PC 是否已存在已经是否达到最大的2个 PC 的限制
                if (!peers.containsKey(from)) {
                    val endPoint = findEndPoint()
                    if (endPoint == MAX_PEER) return@Listener
                    else addPeer(from, endPoint)
                }
                //根据不同的指令类型和数据响应相应步骤的方法
                when (type) {
                    "init" -> createOffer(from)
                    "offer" -> createAnswer(from, payload)
                    "answer" -> setRemoteSdp(from, payload)
                    "candidate" -> addIceCandidate(from, payload)
                }

            } catch (e: JSONException) {
                e.printStackTrace()
            }
        }
        //连接上服务器会返回自己的 clientId 的事件，可开始呼叫。
        val onId = Emitter.Listener { args ->
            val id = args[0] as String
            webrtcListener.onCallReady(id)
        }
		 //获取在线客户端的事件
        val onIdsChanged = Emitter.Listener { args ->
            Log.d(TAG, args.toString())
            val ids = args[0] as JSONArray

            webrtcListener.onOnlineIdsChanged(ids)
        }
    }
    
    //开始 Offer 流程
    private fun createOffer(peerId: String) {
        Log.d(TAG, "CreateOffer")
        val peer = peers[peerId]
        peer?.pc?.createOffer(peer, pcConstraints)
    }

	//开始 Answer 流程
    private fun createAnswer(peerId: String, payload: JSONObject?) {
        Log.d(TAG, "CreateAnswer")
        val peer = peers[peerId]
        val sdp = SessionDescription(
            SessionDescription.Type.fromCanonicalForm(payload?.getString("type")),
            payload?.getString("sdp")
        )
        peer?.pc?.setRemoteDescription(peer, sdp)
        peer?.pc?.createAnswer(peer, pcConstraints)
    }

	//设置 SDP 后无需操作等待 ICE 成功后响应
    private fun setRemoteSdp(peerId: String, payload: JSONObject?) {
        Log.d(TAG, "SetRemoteSDP")
        val peer = peers[peerId]
        val sdp = SessionDescription(
            SessionDescription.Type.fromCanonicalForm(payload?.getString("type")),
            payload?.getString("sdp")
        )
        peer?.pc?.setRemoteDescription(peer, sdp)
    }

	//收到 ICE  后添加到 PC
    private fun addIceCandidate(peerId: String, payload: JSONObject?) {
        Log.d(TAG, "AddIceCandidate")
        val pc = peers[peerId]!!.pc
        if (pc!!.remoteDescription != null) {
            val candidate = IceCandidate(
                payload!!.getString("id"),
                payload.getInt("label"),
                payload.getString("candidate")
            )
            pc.addIceCandidate(candidate)
        }
    }
```

#### 基本流程中的一些细节补充：
* 建立 PeerConnection 时需绑定本地媒体流
```
        init {
            Log.d(TAG, "new Peer: $id $endPoint")
            this.pc = factory.createPeerConnection(iceServers, pcConstraints, this)
            pc?.addStream(localMS!!) //, new MediaConstraints()
            webrtcListener.onStatusChanged("CONNECTING")
        }
```

* 需要实现 `SdpObserver` `PeerConnection.Observer` 接口，用于监听 PeerConnection SDP 关键的回调。

```
	// SDP 创建成功后回调，发送给服务器。
        override fun onCreateSuccess(sdp: SessionDescription) {
            // TODO: modify sdp to use pcParams prefered codecs
            try {
                val payload = JSONObject()
                payload.put("type", sdp.type.canonicalForm())
                payload.put("sdp", sdp.description)
                sendMessage(id, sdp.type.canonicalForm(), payload)
                pc!!.setLocalDescription(this@Peer, sdp)
            } catch (e: JSONException) {
                e.printStackTrace()
            }
        }
       
       // ICE 框架获取候选者成功后的回调，发送给服务器。
        override fun onIceCandidate(candidate: IceCandidate) {
            try {
                val payload = JSONObject()
                payload.put("label", candidate.sdpMLineIndex)
                payload.put("id", candidate.sdpMid)
                payload.put("candidate", candidate.sdp)
                sendMessage(id, "candidate", payload)
            } catch (e: JSONException) {
                e.printStackTrace()
            }

        }
        
        // ICE 连接状态变化时的回调
         override fun onIceConnectionChange(iceConnectionState: PeerConnection.IceConnectionState) {
            webrtcListener.onStatusChanged(iceConnectionState.name)
            Log.d(TAG, "onIceConnectionChange ${iceConnectionState.name}")
            if (iceConnectionState == PeerConnection.IceConnectionState.DISCONNECTED) {
                removePeer(id)
            }
        }
 	
 	//连接成功后，最后获取到媒体流，发给 View 层进行视频/音频的播放。
       override fun onAddStream(mediaStream: MediaStream) {
            Log.d(TAG, "onAddStream " + mediaStream.id)
            // remote streams are displayed from 1 to MAX_PEER (0 is localStream)
            webrtcListener.onAddRemoteStream(mediaStream, endPoint + 1)
        }
	
	//媒体流断开
        override fun onRemoveStream(mediaStream: MediaStream) {
            Log.d(TAG, "onRemoveStream " + mediaStream.id)
            removePeer(id)
        }
        
```

在`onAddStream`中将 MediaStream 发给 View 层后 WebRtcClient 中的连接的工作基本完成。

* View 层中将 MediaStream 绑定到 View 中

```
	//使用 org.webrtc.SurfaceViewRenderer
        <org.webrtc.SurfaceViewRenderer
            android:id="@+id/remote_renderer"
            android:layout_width="match_parent"
            android:layout_height="match_parent" />
            
       //初始化
     override fun onCreate(savedInstanceState: Bundle?) {      
        binding.remoteRenderer.apply {
            setEnableHardwareScaler(true)
            init(eglBase.eglBaseContext, null)
        }
      }
	//绑定从 WebRtcClent 中转发 MediaStream
      override fun onAddRemoteStream(remoteStream: MediaStream, endPoint: Int) {
                    remoteStream.videoTracks[0].addSink(binding.remoteRenderer)
                }
```

此外，上面只是展示了关键步骤，但实际编码中回调较多，还是比较繁杂。
完整代码参考 [https://github.com/xiejinpeng007/WebRTC-Android-Server](https://github.com/xiejinpeng007/WebRTC-Android-Server)
### 信令服务器端（NodeJS）

负责转发信令等功能  

部署：
在 SignalServer 根目录下执行 `node app.js`  会部署在 3000 端口，并监听客户端的连接情况。

<img src="/Users/jinpengxie/Desktop/webrtc_server.png" height="400">

### 使用和演示

输入信令服务器地址（公网和局域网皆可）连接服务器后， 根据在线用户进行呼叫，由于 STUN 服务器用了 Google 的，所以需要梯子。

1. 设定服务器地址查看在线用户  
<img src="/Users/jinpengxie/Desktop/webrtc_android.png" height="500">

2. 选择用户进行拨号连接  
<img src="/Users/jinpengxie/Desktop/webrtc_android_demo.gif" height="500">

### 总结
优点：

* 当然是大部分流量不经过服务器直接点对点(P2P)传输，可以大大的节省服务商的带宽资源。

缺点:

* 原生只支持1对1的通信，要实现多人通信需要借助服务端的其它方案例如中转。
* 复杂的网络场景连接质量无法保证，比如跨国等情况，也需要服务商进行优化。

大多使用 WebRTC 技术的都根据具体业务都在此基础上进行了二次封装， Google 自家应用上也看到在使用相关的技术，所以总的来说 WebRTC 确实是一套实际可用的技术。


### 参考:
[https://github.com/xiejinpeng007/WebRTC-Android-Server](https://github.com/xiejinpeng007/WebRTC-Android-Server)  (Demo)  
[https://webrtc.org/native-code/android/](https://webrtc.org/native-code/android/)  
[https://developer.mozilla.org/en-US/docs/Web/API/WebRTC_API/Protocols](https://developer.mozilla.org/en-US/docs/Web/API/WebRTC_API/Protocols)
