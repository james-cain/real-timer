# real-timer

> 该仓库用来记录学习webrtc的心路历程

[WebRTC相关资料和教程](https://webrtc.org/)

[官方示例程序AppRTC](https://github.com/webrtc/apprtc)

[在线demo](https://github.com/webrtc/samples)

[WebRTC-Experiment](https://github.com/muaz-khan/WebRTC-Experiment)

WebRTC，全称为Web Real-Time Communication(Web实时通信)。它是基于UDP传输的，用来实现浏览器之间端到端的音频、视频及数据共享。WebRTC让实时通信更加简单，不需要借助第三方插件和软件。

WebRTC让浏览器具备了完整功能的音频和视频引擎，帮助我们解决了实时解码音频和视频流的困难性，并适应网络抖动和时延。具体引擎实现的功能如下：

![webrtc-peer-to-peer-imx6](http://www.reyshieh.com/assets/webrtc-peer-to-peer-imx6.jpg)

获得的音频流要经过降噪和回声消除处理，然后自动通过优化的窄带或宽带音频编解码器编码，还要通过特殊的错误补偿算法消除网络抖动和丢包造成的损失。

获得的视频流更着重与影响品质，选择最优的压缩和编解码方案，应用抖动和丢包补偿等。

实时传输讲究的就是**及时和低延迟**。往往音频和视频流传输途中会出现丢包现象，音频和视频编解码可以填充小的数据空白，通常对输出品质的影响很小。另外，应用也必须实现自己的逻辑，以便因传输其他应用数据而丢包或延迟时快速恢复。

因此，**UDP协议就是传输实时数据的首选协议**。是不是拥有了UDP协议就可以很正常的发送音频、视频了？其实并不然，正因UDP并不像TCP协议有一套完整的握手协议，使得要想让数据穿透层层NAT和防火墙并不是简单的事情，且WebRTC要求，**每个流都要经过协商设定参数，对用户数据要加密，并要实现拥塞和流量控制等**。为了实现要求，浏览器还需要提供大量协议和服务的支持。协议封层如下：

![diagram_2_en.png](http://www.reyshieh.com/assets/diagram_2_en.png)

- ICE、STUN、TURN是通过UDP建立并维护端到端连接必须提供的

  ![webrtc-connection-ice-types.jpg](http://www.reyshieh.com/assets/webrtc-connection-ice-types.jpg)

- DTLS用于**保障传输数据的安全，加密**。DTLS本质上就是TLS，只是为了兼容UDP的数据报传输而做了一些微小的修改。它针对握手顺序实现了一个"mini TCP"。具体通过在每条握手记录中添加**分段偏移字段和序号**，满足了有序交付的条件，也让大记录可以被分段成多个分组并在另一端进行组装。还要**处理丢包问题**，两端都**使用计时器，如果预定时间内没有收到应答，就重传握手记录**。WebRTC客户端自动为每一端生成自己签名的证书，因此不需要证书链验证。

- SRTP负责把**数字化的音频采样和视频帧用一些元数据封装起来，以辅助接收方处理这些流**。每个SRTP分组都包含一个自动递增的序号，以便接收端检测和发现媒体数据是否乱序；每个SRTP分组都包含一个时间戳，表示媒体第一字节的采样时间，用于媒体流的同步；都包含加密的媒体净荷，以及可选的认证标签，用于验证分组的完整性

- SCTP专门是为传输任意应用数据的DataChannel API而设计的，SCTP在两端之间建立的DTLS信道之上运行。**SCTP是传输层协议**，直接在IP协议之上运行。特点是结合了UDP和TCP的优点，**面向消息的API、支持多路复用、可配置的可靠性及交付语义，内置流量和拥塞控制机制**。实际和HTTP2.0的二进制分帧层很类似，首部包含12位公共首部和16位的一或多个控制字段或数据块组成

接下来，介绍建立端到端连接的过程。

WebRTC两端可能位于同一局域网内，也有可能分别位于自己的私有网络中。对于第一种情况相对来说简单些，第二种情况，并不会知道中间隔着多少层NAT。为了发起会话，首先必须找到两端的候选IP和端口(candidate)，穿越NAT，然后检查连接，从中找到可用路径。但这一步即使找到了，也不一定能成功。原因就是，远端可能不在线或根本不能访问到，或根本不想与其他端建立连接。

1. 发信号和协商会话

   在检查连接或协商会话之前，必须知道能否将信息发送到另一端，以及另一端是否愿意建立连接。对此，需要一个**共享的发信通道**。

   WebRTC把发送信号和协议的选择交给应用，可以让现有通信设施中的其他发信协议操作实现。

   如：

   - SIP(Session Initiation Protocol，会话初始协议) - 广发用于通过IP实现的语音通话(VoIP)和视频会议。要使WebRTC网页应用能和其他视频会议系统的SIP客户端通讯，WebRTC需要一个代理服务器做中介信令。信令需要流过网关，但是一旦通信已经建立起来，SRTP就可以点对点传输。
   - Jingle - XMPP协议的发信扩展，用于通过IP实现的语音通话(VoIP)和视频会议的会话控制
   - ISUP(ISDN User Part，ISDN用户部分) 

   WebRTC应用可以选择已有的任何**发信协议和网关**，利用既有通信系统协商一次通话或视频会议

   ![signaling.svg](http://www.reyshieh.com/assets/signaling.svg)

   发信服务器可以作为已有通信网络的网关，此时由网络负责将连接提议发送给目标端，然后再将应答返回给WebRTC客户端，以初始化信息交换。

2. 会话描述协议(Session Description Protocol，SDP)

   SDP描述端到端连接的参数。SDP不包含媒体本身的任何信息，仅用于描述"会话状况"，表现为一系列的连接属性：**要交换的媒体类型(音频、视频及应用数据)、网络传输协议、使用的编解码器及其设置、带宽及其他元数据**。

   **假设应用实现了共享的发信通道，接下来可以执行发起WebRTC连接的初始步骤**。

   ```js
   var signalingChannel = new SignalingChannel(); // 初始化共享的发信通道
   var pc = new RTCPeerConnection({}); // 初始化RTCPeerConnection对象
   
   navigator.getUserMedia({ "audio": true }, gotStream, logError); // 向浏览器请求音频流
   
   function gotStream(stream) {
       pc.addstream(stream); // 通过RTCPeerConnection注册本地音频流
       
       pc.createOffer(function(offer) { // 创建端到端连接的SDP描述
           pc.setLocalDescription(offer); // 已生成的SDP作为端到端连接的本地描述
           signalingChannel.send(offer.sdp); // 通过发信通道向远端发送SDP提议
       });
   }
   
   function logError() { ... }
   ```

   对于上例中，createOffer生成有关会话的SDP描述具有以下信息：

   ```js
   // ... 省略 ...
   m=audio 1 RTP/SAVPF 111 ... // 带反馈的安全音频信息
   a=extmap: 1 urn:ietf:params:rtp-hdrext:ssrc-audio-level
   a=candidate:1862263974 1 udp 2113937151 192.168.1.73 60834 typ host ... // 媒体流的候选IP、端口及协议
   a=mid:audio
   a=rtpmap:111 opus/48000/2 // Opus编解码及基本配置
   a=fmtp:111 minptime=10
   // ...省略内容 ...
   ```

   要建立端到端的连接，两端都必须遵循一个对称的工作流，以交换各自音频、视频及其他数据流的SDP描述。

   在通过发信通道交换SDP会话描述后，双方就交换了经过协商的流类型及相应设置。

   **在此之前，还必须注意连接检查和NAT穿越**。

3. 执行路由和连接检查(ICE)

   建立端到端的连接，需要解决多层防火墙和NAT设备阻隔问题。

   第一种情况，两端位于同一个内部网中，而且之间不存在防火墙或NAT设备。此时，要建立连接，两端只要查询操作系统获知IP地址，将IP地址加端口号追加到生成的SDP字符串中，再把SDP转发给另一端即可。SDP交换一完成，两端就可以发起直接的端到端连接。

   另一种情况就是之前提到的，位于两个不同的局域网，重复以上流程显然是不可能成功的。因此，需要一条连接两端的公共路由线路。但不用惆怅，WebRTC框架代替做了大部分复杂的工作，包括

   - **每个RTCPeerConnection连接对象都包含一个"ICE代理"**
   - ICE代理负责**收集IP地址和端口(candidate)**
   - ICE代理负责**执行两端的连接检查**
   - ICE代理负责**发送连接持久化信息**

   无论是本地还是远程，都会使用这个工作流程来操作。本地ICE代理会自动开始发现本地端所有可能的候选IP和端口的进程：

   - ICE代理向操作系统查询本地IP地址
   - 如果有配置，ICE代理会查询外部STUN服务器，已取得本地端的公共IP和端口号
   - 如果有配置，ICE代理会将TURN服务器追加到最后一个候选项；假如端到端的连接失败，数据将通过指定的中间设备转发

   每发现一个新候选项(一个IP加一个端口号)。代理就会自动通过RTCPeerConnection对象注册它，并通过一个回调函数(onicecandidate)通知应用。ICE在完成收集工作后，也会再触发同一个回调函数，以通知应用。

   ```js
   var ice = {
       "iceServers": [
           { "url": "stun.stun.l.google.com:19302" }, // STUN服务器
           { "url": "turn.user@turnserver.com", "credential": "pass" } // TURN服务器
       ]
   };
   
   var signalingChannel = new SignalingChannel();
   var pc = new RTCPeerConnection(ice);
   
   navigator.getUserMedia({ "audio": true }, gotStream, logError);
   
   function gotStream(stream) {
       pc.addStream(stream);
       
       pc.createOffer(function(offer) {
           pc.setLocalDescription(offer); // 初始化ICE收集过程
       });
   }
   
   pc.onicecandidate = function(evt) {
       if (evt.target.iceGatheringState == "complete") { // 预订ICE事件，监听ICE收集完成
           local.createOffer(function(offer) {
               console.log("Offer with ICE candidates: " + offer.sdp);
               signalingChannel.send(offer.sdp);
           });
       }
   }
   
   // ...
   
   // 包含ICE候选项的提议：
   // a=candidate:1862263974 1 udp 2113937151 192.168.1.73 60834 typ host ...
   ```

ICE收集过程是自动触发的，STUN查找是在后台执行的，而发现的候选项也会自动通过RTCPeerConnection对象注册。另一端接收到ICE候选项后，就可以进行第二步--**建立端到端的连接了：只要RTCPeerConnection对象设置了远程会话描述(包含另一端的一组候选IP和端口号)，ICE代理就会执行连接检查，以确定能否抵达另一端**。

ICE代理发送消息(STUN绑定请求)，另一端接收之后必须以一个成功的STUN响应确认。如果这个过程完成，那么就代表着有了一条端到端连接的路由线路；相反，如果所有候选项都绑定失败，要么将RTCPeerConnection标记为失败，要么回退到靠TURN转发服务器建立连接。

上面提到的ICE确认方式其实是很耗时的，为了减少初始化端到端连接的时间，可以通过**端到端之间的增量收集和连接检查方式**(增量提供，Trickle ICE)来处理。原理如下：

- 两端交换没有ICE候选项的SDP提议
- 发现ICE候选项之后，通过发信通道发送到另一端
- 新候选描述一就绪，立即执行ICE连接检查

也就是不等ICE收集过程完成，依靠发信通道向另一端递增地交付更新，从而加快协商。代码如下

```js
var ice = {
    "iceServers": [
        { "url": "stun.stun.l.google.com:19302" }, // STUN服务器
        { "url": "turn.user@turnserver.com", "credential": "pass" } // TURN服务器
    ]
};

var signalingChannel = new SignalingChannel();
var pc = new RTCPeerConnection(ice);

navigator.getUserMedia({ "audio": true }, gotStream, logError);

function gotStream(stream) {
    pc.addStream(stream);
    
    pc.createOffer(function(offer) {
        pc.setLocalDescription(offer); // 初始化ICE收集过程
        signalingChannel.send(offer.sdp); // 发送不包含ICE候选项的SDP提议
    });
}

pc.onicecandidate = function(evt) {
    if (evt.candidate) {
        signalingChannel.send(evt.candidate); // 本地ICE代理发现一个ICE候选项就立即发送
    }
}

signalingChannel.onmessage = function(msg) {
    if (msg.candidate) {
        pc.addIceCandidate(msg.candidate); // 建立远程ICE候选项并开始连接检查
    }
}
```

ICE框架中存在两种连接状态，分别为`iceGatheringState`和`iceConnectionState`。

其中`iceGatheringState`属性中保存的是本地端候选项的收集状态，可能有三个值：

- new: 对象刚刚创建，还没有连网
- gathering: ICE代理正在收集本地候选项
- complete: ICE代理收集过程完成

`iceConnectionState`属性中保存着端到端的连接状态，可能有7个值：

- new: ICE代理正在收集候选项且/或正在等待远程候选项的到来
- checking: ICE代理至少已经收到来自一个组件的远程候选项
- connected: ICE代理已经找到一条通过所有组件的可用连接，但仍在检查更好的连接，此时仍有可能还在收集
- completed: ICE代理已经完成收集和检查，发现了通过所有组件的连接
- failed: ICE代理检查至少有一个组件的连接失败
- disconnected: 一或多个组件的活动检查失败，相对failed更严重
- closed: ICE代理关闭，不再响应STUN请求

![iceconnectionstate](http://www.reyshieh.com/assets/iceConnectionState.png)

## JavaScript API

### getUserMedia()

可以通过该API获取到音频和视频流，并对它们进行操作和处理。同时该API可以指定一系列强制和可选的约束条件，匹配应用的需求。

```html
<video autoplay></video>
```

```js
<script>
	var constraints = {
        audio: true, // 指定音频轨道
        video: { // 指定视频轨道
            mandatory: { // 对视频轨道的强制约束条件
                width: { min: 320 },
                height: { min: 180 }
            },
            optional: [ // 对视频轨道的可选约束条件
                { width: { max: 1280 } },
                { frameRate: 30 },
                { facingMode: "user" }
            ]
        }
	}
	navigator.getUserMedia(constraints, gotStream, logError); // 从浏览器中请求音频和视频流
	
	function gotStream(stream) {
        var video = document.querySelector('video');
        video.src = window.URL.createObjectURL(stream);
	}

	function logError(error) { ... }
</script>
```

当获取完流以后，可以将它提供给其他浏览器API：

- 通过Web Audio API在浏览器中处理音频
- 通过Canvas API采集个别视频帧并加以处理
- 通过CSS3和WebGL API为输出的流应用各种2D/3D特效

> 注：新版使用Navigator.mediaDevices.getUserMedia()取代navigator.getUserMedia()

constraints配置可以参考[MediaTrackConstraints 规范文档](https://w3c.github.io/mediacapture-main/getusermedia.html#media-track-constraints)

新版e.g.

```js
const mediaStreamContraints = {
    video: true
};

const localVideo = document.querySelector('video');

let localStream;

function gotLocalMediaStream(mediaStream) {
    localStream = mediaStream;
    localVideo.srcObject = mediaStream;
}

function handleLocalMediaStreamError(error) {
    console.log('navigator.getUserMedia error: ', error);
}

navigator.mediaDevices.getUserMedia(mediaStreamConstranits)
	.then(gotLocalMediaStream)
	.catch(handleLocalMediaStreamError);
```

### MediaStream

MediaStream接口是一个实时媒体内容的流，以便应用代码从中取得数据，操作个别的轨道和控制输出。所有的音频和视频处理，比如降噪、均衡、影像增强等都由音频和视频引擎自动完成。

一个流包含多个轨道，比如视频和音频轨道。

多个轨道之间是相互同步的。

输入源可以是物理设备，如麦克风、摄像头、用户硬盘或另一端服务器中的文件。

输出可以被发送到疑惑多个目的地，本地的视频或音频元素、后期处理的JavaScript代理或远程另一端。

### RTCPeerConnection

RTCPeerConnection接口负责维护每一个端到端连接的完整生命周期：

- 管理穿越NAT的完整ICE工作流
- 发送自动(STUN)持久化信号
- 跟踪本地流
- 跟踪远程流
- 按需触发自动流协商
- 提供必要的API，生成连接提议，接收应答，允许查询连接的当前状态等

RTCPeerConnection把所有连接设置、管理和状态都封装在了一个接口中。

RTCPeerConnection构造函数可以传入配置参数来配置新的连接，具体配置可以参考[MDN RTCPeerConnection](https://developer.mozilla.org/en-US/docs/Web/API/RTCPeerConnection/RTCPeerConnection)中的RTCConfiguration dictionary

### RTCDataChannel

RTCDataChannel支持端到端的任意应用数据交换。建立RTCPeerConnection连接之后，**两端可以打开一或多个信道交换文本或二进制数据**：

```js
function handleChannel(chan) { // 在DataChannel对象上注册类似WebSocket的回调
    chan.onerror = function(error) {}
    chan.onclose = function() {}
    chan.onopen = function(evt) {
        chan.send("DataChannel connection established. Heelo peer!");
    }
    chan.onmessage = function(msg) {
        if (msg.data instanceof Blob) {
            processBlob(msg.data);
        } else {
            processText(msg.data);
        }
    }
}

var signalingChannel = new SignalingChannel();
var pc = new RTCPeerConnection(iceConfig);

var dc = pc.createDataChannel("namedChannel", { reliable: false }); // 以最合适的交付语义初始化新的DataChannel

handleChannel(dc); // 在本地初始化的DataChannel上注册回调
pc.onDataChannel = handleChannel; // 在远端初始化的DataChannel上注册回调
```

RTCDataChannel API 是有意照搬WebSocket，每个信道都会触发同样的onerror、onclose、onopen和onmessage回调，而且每个信道也会提供同样的binaryType、bufferedAmount和protocol字段。

RTCDataChannel使用的是SCTP协议。因此，初始端在生成连接提议，或者另一端生成应答时，它们都会特意在生成的SDP字符串中包含SCTP关联的参数：

```js
// (... 省略内容 ...)
m=application 1 DTLS/SCTP 5000 // 告知对方想使用DTLS之上的SCTP
c=IN IP4 0.0.0.0 // 0.0.0.0候选项表示使用增量ICE
a=mid:data
a=fmtp:5000 protocol=webrtc-datachannel; streams=10 // SCTP之上的RTCDataChannel协议，最多10个并行流
// (... 省略内容 ...)
```

沟通完信道参数，两端就可以交换应用数据了。本质上，每个信道都还是作为一个独立的SCTP流发送数据，即所有信道都是在同一个SCTP关联之上多路复用出来的。**这样可以避免不同流之间的队首阻塞，在同一个SCTP关联上同时打开多个信道。**

## 实例

实例伪代码提取自[webrtc-web](https://github.com/googlecodelabs/webrtc-web)，需要完整代码，可以自行点击进入阅读。

如果需要对低版本浏览器做兼容，强烈建议使用[webrtc-adapter](https://github.com/webrtc/adapter)，shim将应用程序与规范更改和前缀差异隔离开来。

### 用例1：本地读取视频流(MediaStream)

```js
let localStream;

// 获取媒体流传入参数，只获取video
const mediaStreamContraints = {
    video: true
};

// 初始化媒体流
navigator.mediaDevices.getUserMedia(mediaStreamContraints)
	.then(getLocalMediaStream).catch(handleLocalMediaStreamError);

// 成功回调，添加媒体流到video标签
function getLocalMediaStream(mediaStream) {
    localStream = mediaStream;
    localStream.srcObject = mediaStream;
}

// 错误回调，记录错误日志
function handleLocalMediaStreamError(error) {
    console.log('navigator.getUserMedia error:', error);
}
```

### 用例2：本地端到端建立连接(不使用STUN、TURN)

WebRTC客户端之间创建视频通话，首先每个客户端要创建一个`RTCPeerConnection`实例，通过`getUserMedia()`获取本地媒体流；其次ICE执行路由和检查连接，将所有可能的连接点都当做ICE候选并发送给对方，获取本地及远程的描述信息(SDP)，相互确认连接建立完成；顺利开始传输流。期间可能会遇到连接断开、找到更优的路径等，都是由内置ICE实现切换和重连。

1. 调用getUserMedia()，获取到本地stream传给localVideo

   ```js
   navigator.mediaDevices.getUserMedia(mediaStreamConstraints).
     then(gotLocalMediaStream).
     catch(handleLocalMediaStreamError);
   
   function gotLocalMediaStream(mediaStream) {
     localVideo.srcObject = mediaStream;
     localStream = mediaStream;
     trace('Received local stream.');
     callButton.disabled = false;  // Enable call button.
   }
   ```

2. 首先创建RTCPeerConnection对象

   ```js
   let localPeerConnection;
   let servers = null;
   localPeerConnection = new RTCPeerConnection(servers);
   ```

   servers参数为null，可以指定STUN和TURN服务器相关的信息。

3. 设置onicecandidate回调，本地ICE代理在发现一个ICE候选项后就立即发送(**此处采用的是增量提供的方式，先用createOffer/createAnswer建立端到端连接的SDP(提议)描述，再等候选描述就绪，立即执行ICE连接检查**)。因为只有本地直接通信，不再需要外部消息服务，无论是local peer还是remote peer，都只要调用`addIceCandidate()`方法，建立远程ICE候选项并开始连接检查。

   ```js
   localPeerConnection.addEventListener('icecandidate', handleConnection);
   localPeerConnection.addEventListener('iceconnectionstatechange', handleConnectionChange);
   
   function handleConnection(event) {
       const peerConnection = event.target;
       const iceCandidate = event.candidate;
       
       if(iceCandidate) {
           const newIceCandidate = new RTCIceCandidate(iceCandidate);
           const otherPeer = getOtherPeer(peerConnection);
           
           otherPeer.addIceCandidate(newIceCandidate)
               .then(() => {
               // successcallback
           }).catch((error) => {
               // failcallback
           });
       }
   }
   ```

4. 通过RTCPeerConnection注册本地流，remote peer通过监听`onaddstream`获取到远程过来的流，并进行下一步操作，无论是输出到对应的`audio/video`，还是`canvas API`对帧处理，又或者是`WebGL`处理

   ```js
   localPeerConnection.addStream(localStream);
   remotePeerConnection.addEventListener('addstream', gotRemoteMediaStream);
   
   function gotRemoteMediaStream(event) {
       const mediaStream = event.stream;
       remoteVideo.srcObject = mediaStream;
       remoteStream = mediaStream;
   }
   ```

5. webRTC客户端通过`createOffer`和`createAnswer`交换SDP提议，其中SDP包括本地和远程音频/视频媒体信息，如要交换的媒体类型（音频、视频及应用数据）、网络传输协议、使用的编解码其及其设置、带宽及其他元数据，(以下流程中local peer用A表示，remote peer用B表示)

   首先，A先用`setLocalDescription()`方法将本地会话信息保存，接着通过信令通道，将这些信息发送给B；其次，B使用`setRemoteDescription()`方法将A传过来的远端会话信息填进去；然后B执行`createAnswer()`方法，传入获取到的远端会话信息，生成一个与A适配的本地会话，用`setLocalDescription()`方法保存，也发送给A；最后，A获取到B的会话描述信息之后，使用`setRemoteDescription()`方法将远端会话信息设置进去。

   ```js
   // Logs offer creation and sets peer connection session descriptions.
   function createdOffer(description) {
     trace(`Offer from localPeerConnection:\n${description.sdp}`);
   
     trace('localPeerConnection setLocalDescription start.');
     localPeerConnection.setLocalDescription(description)
       .then(() => {
         setLocalDescriptionSuccess(localPeerConnection);
       }).catch(setSessionDescriptionError);
   
     trace('remotePeerConnection setRemoteDescription start.');
     remotePeerConnection.setRemoteDescription(description)
       .then(() => {
         setRemoteDescriptionSuccess(remotePeerConnection);
       }).catch(setSessionDescriptionError);
   
     trace('remotePeerConnection createAnswer start.');
     remotePeerConnection.createAnswer()
       .then(createdAnswer)
       .catch(setSessionDescriptionError);
   }
   
   // Logs answer to offer creation and sets peer connection session descriptions.
   function createdAnswer(description) {
     trace(`Answer from remotePeerConnection:\n${description.sdp}.`);
   
     trace('remotePeerConnection setLocalDescription start.');
     remotePeerConnection.setLocalDescription(description)
       .then(() => {
         setLocalDescriptionSuccess(remotePeerConnection);
       }).catch(setSessionDescriptionError);
   
     trace('localPeerConnection setRemoteDescription start.');
     localPeerConnection.setRemoteDescription(description)
       .then(() => {
         setRemoteDescriptionSuccess(localPeerConnection);
       }).catch(setSessionDescriptionError);
   }
   ```

### 用例3: RTCDataChannel传输数据

WebRTC可以通过数据通道(data channel)，实现端到端的数据传输。

RTCDataChannel采用的是SCTP应用层协议，该协议类似于HTTP2.0方式，使用二进制流，多路复用传输。但是它不能作为TCP或UDP的替代品，只用于WebRTC中。

使用的方式和音视频的传输类似。区别为：

- 音视频local peer通过RTCPeerConnection的`addStream`注册本地流，remote peer通过RTCPeerConnection的`onaddstream`事件监听获取流
- 数据local peer通过RTCPeerConnection的`createDataChannel`初始化DataChannel，并在初始化完成的channel上注册事件，如`onopen`、`onclose`，注册的事件的api和websocket一致，要往remote peer传输数据时，调用DataChannel的`send()`传输数据，remote peer通过RTCPeerConnection的`ondatachannel`事件监听数据以及注册事件，如`onmessage`、`onopen`、`onclose`

该例中的前面几步流程与例2中的1.2.3.5一致，创建RTCPeerConnection对象，交换SDP提议，获取ICE候选项，基础设施搭建好后才开始正式的传输流程，下面只对传输流程描述，从第5开始。

1-4. 与例2中的相似

5. local peer创建`DataChannel`并注册事件，remote peer监听`ondatachannel`，注册事件等待对端的数据传输过来

   ```js
   var sendChannel;
   sendChannel = localConnection.createDataChannel('sendDataChannel', dataConstraint);
   sendChannel.onopen = onSendChannelStateChange;
   sendChannel.onclose = onSendChannelStateChange;
   
   function onSendChannelStateChange() {
       var readyState = sendChannel.readyState;
       if (readyState === 'open') {
           // 传输数据前要做的操作
       } else {
           // 传输数据结束后要做的操作
       }
   }
   
   remoteConnection.ondatachannel = receiveChannelCallback;
   function receiveChannelCallback(event) {
       receiveChannel = event.channel;
       receiveChannel.onmessage = onReceiveMessageCallback;
       receiveChannel.onopen = onReceiveChannelStateChange;
       receiveChannel.onclose = onReceiveChannelStateChange;
   }
   function onReceiveMessageCallback(event) {
       // 接收到远端数据后做的操作
   }
   function onReceiveChannelStateChange() {
   }
   ```

6. local peer通过RTCDataChannel的send()方法与发送数据

   ```js
   function sendData(data) {
       sendChannel.send(data);
   }
   ```

需要注意的是，createDataChannel带有第二个参数`dataConstraint`，可以通过配置该参数，来传递各种类型特征的数据，如，可靠性优先还是效率优先，可以参考[MDN createDataChannel](https://developer.mozilla.org/en-US/docs/Web/API/RTCPeerConnection/createDataChannel)中的RTCDataChannelInit dictionary部分。

- ordered - 是否需要按照顺序到达目的端，true为按照顺序，false可以无序，默认为`true`
- maxPacketLifeTime - 尝试在不可靠模式下传输消息的最大毫秒数，默认为`null`
- maxRetransmits - 在不可靠模式下，用户代理在首次接收失败后重传数据的最大次数，默认为`null`
- protocol - RTCDataChannel下使用的子协议名称 默认为空字符串，最大长度不能尝过65535
- negotiated - 默认`false`，数据通道在带内协商，一端调用createDataChannel，另一端用ondatachannel事件监听；若为true，在带外协商，在这种情况下，双方使用一致同意的id调用createDataChannel
- Id - 通道的id标识，若不填，用户代理会选择一个id填入

### 用例4: 配置信令服务

信令传输(signaling)：传输流媒体音视频/数据，必须先相互交换元数据信息，包括

- 候选网络信息(candidate)
- 媒介相关的邀请信息(offer)和响应信息(answer)，比如分辨率、编解码器等

搭建信令服务器(signaling server)，为WebRTC客户端(peers)之间传递消息，实际上这些信令都是纯文本格式的，也就是将JavaScript对象序列化为字符串的形式(stringified)。

在实际应用中，还是需要使用STUN和TURN服务器支持的。点击[参考链接](<https://www.html5rocks.com/en/tutorials/webrtc/infrastructure/>)

> 该例用Node.js搭建信令服务器，用Socket.IO模块和JavaScript库来传递消息。
>
> Socket.IO非常适合用于学习WebRTC信令，内置了rooms概念。

对于商业级产品来说，可以有更多更好的选择，点击[参考链接 How to Select a Signaling Protocol for Your Next WebRTC Project](https://bloggeek.me/siganling-protocol-webrtc/)

用例中关键步骤有

1. 客户端先发起`create or join`事件，由服务端来判断在客户端发起之前是否已经有别的客户端发起等待别的加入

   ```js
   var socket = io.connect();
   
   sokect.emit('create or join', room); // 向服务端提交请求
   ```

2. 服务端接收到请求，例子只是两人之间的通信，因此可以使用socket.rooms API判断是否已经满足要求，并向服务端发送响应

   ```js
   socket.on('create or join', function(room) {
       var clientsInRoom = io.sockets.adapter.rooms[room];
       var numClients = clientsInRoom ? Object.keys(clientsInRoom.sockets).length : 0;
       if (numClients === 0) {
           socket.join(room);
           socket.emit('created', room, socket.id);
       } else if (numClients === 1) {
           io.sockets.in(room).emit('join', room);
           socket.join(room);
           socket.emit('joined', room, socket.id);
           io.sockets.in(room).emit('ready');
       } else {
           socket.emit('full', room);
       }
   });
   ```

### 用例5: 集成对等通信和信令服务

在用例4的基础上，先搭建信令服务，打开的客户端都会先调用`getUserMedia()`获取本地流，并放入本地video中。结合用例2分析：

1. 客户端(A)打开页面，向信令服务发送`got user media`消息，此时因为只有一个客户端打开，没有远端通信，因此只能看见本地流视频

   ```js
   navigator.mediaDevices.getUserMedia({
     audio: false,
     video: true
   })
   .then(gotStream)
   .catch(function(e) {
     alert('getUserMedia() error: ' + e.name);
   });
   
   function gotStream(stream) {
     console.log('Adding local stream.');
     localStream = stream;
     localVideo.srcObject = stream;
     sendMessage('got user media');
     if (isInitiator) { // 只有再次打开另一个客户端，该值改为true，才开始交换sdp，以及candidate
       maybeStart();
     }
   }
   ```

2. 再次打开一个客户端(B)，开始建立端到端连接前的检查(`createOffer/createAnswer`)，交换`SDP`，添加流(`addStream`)，接下来监听candidate，一般情况下各端都会产生两个candidate，然后添加到远端的candidate中(`pc.addIceCandidate(candidate)`)，最后全部完成，开始正常的流传输通信。

   ```js
   socket.on('message', function(message) {
     console.log('Client received message:', message);
     if (message === 'got user media') {
       maybeStart();
     } else if (message.type === 'offer') {
       if (!isInitiator && !isStarted) {
         maybeStart();
       }
       pc.setRemoteDescription(new RTCSessionDescription(message));
       doAnswer();
     } else if (message.type === 'answer' && isStarted) {
       pc.setRemoteDescription(new RTCSessionDescription(message));
     } else if (message.type === 'candidate' && isStarted) {
       var candidate = new RTCIceCandidate({
         sdpMLineIndex: message.label,
         candidate: message.candidate
       });
       pc.addIceCandidate(candidate);
     } else if (message === 'bye' && isStarted) {
       handleRemoteHangup();
     }
   });
   
   function doCall() {
     console.log('Sending offer to peer');
     pc.createOffer(setLocalAndSendMessage, handleCreateOfferError);
   }
   
   function doAnswer() {
     console.log('Sending answer to peer.');
     pc.createAnswer().then(
       setLocalAndSendMessage,
       onCreateSessionDescriptionError
     );
   }
   ```

### 用例6: 拍照并传给对方

该用例是用例3的扩展，实现实时捕捉最新视频最新帧，传送给远端。

1. 在点击Snap按钮，会从video流中把最新帧捕获下来，并通过canvas元素展示

   ```js
   function snapPhoto() {
     photoContext.drawImage(video, 0, 0, photo.width, photo.height);
     show(photo, sendBtn);
   }
   ```

2. 点击Send按钮，会将图像转换成字节数组(bytes)，分块(chunk)的方式发送到对端

   ```js
   function sendPhoto() {
     // 将数据分块的字节数长度;
     var CHUNK_LEN = 64000;
     var img = photoContext.getImageData(0, 0, photoContextW, photoContextH),
       len = img.data.byteLength,
       n = len / CHUNK_LEN | 0;
   
     console.log('Sending a total of ' + len + ' byte(s)');
     dataChannel.send(len);
   
     // split the photo and send in chunks of about 64KB
     for (var i = 0; i < n; i++) {
       var start = i * CHUNK_LEN,
         end = (i + 1) * CHUNK_LEN;
       console.log(start + ' - ' + (end - 1));
       dataChannel.send(img.data.subarray(start, end));
     }
   
     // send the reminder, if any
     if (len % CHUNK_LEN) {
       console.log('last ' + len % CHUNK_LEN + ' byte(s)');
       dataChannel.send(img.data.subarray(n * CHUNK_LEN));
     }
   }
   ```

3. 对端接收到字节，拼接字节，转为图像存入canvas中

   ```js
   function receiveDataChromeFactory() {
     var buf, count;
   
     return function onmessage(event) {
       if (typeof event.data === 'string') {
         buf = window.buf = new Uint8ClampedArray(parseInt(event.data));
         count = 0;
         console.log('Expecting a total of ' + buf.byteLength + ' bytes');
         return;
       }
   
       var data = new Uint8ClampedArray(event.data);
       buf.set(data, count);
   
       count += data.byteLength;
       console.log('count: ' + count);
   
       if (count === buf.byteLength) {
         // we're done: all data chunks have been received
         console.log('Done. Rendering photo.');
         renderPhoto(buf);
       }
     };
   }
   
   function renderPhoto(data) {
     var canvas = document.createElement('canvas');
     canvas.width = photoContextW;
     canvas.height = photoContextH;
     canvas.classList.add('incomingPhoto');
     // trail is the element holding the incoming images
     trail.insertBefore(canvas, trail.firstChild);
   
     var context = canvas.getContext('2d');
     var img = context.createImageData(photoContextW, photoContextH);
     img.data.set(data);
     context.putImageData(img, 0, 0);
   }
   ```

## 遗留问题

看似经过这几个用例我们已经可以写出创造一个类似视频聊天的room，同时还能传输流数据。但是，这些用例都只是在本机里实现的，而无法在局域网内，甚至更不用提公网了。

对于STUN、TURN还存在很多未知疑惑。还要投入更多的时间去研究这块内容。

贴一个连接后面阅读http://www.52im.net/thread-557-1-1.html

## 信令

信令即协调通讯的过程。WebRTC应用要发起一个对话，客户端需要交换如下信息：

- 用于打开和关闭通讯的会话信息
- 错误信息
- 媒体元数据如编解码器及其设置，带宽和媒体类型
- 秘钥数据，用于创建安全连接
- 网络数据，如外部能访问的主机IP和端口

信令过程需要客户端之间能来回传递消息，但实际上，这个机制WebRTC是不提供的，需要自己创建。上面的用例4开始就有介绍了一种搭建信令服务的方式--`WebSocket`，除了这个方式，还可以使用以前的比较旧的技术，如长轮询、用EventSource API从服务器端推送，和XHR结合构建成交换信令消息的服务器，即一个从呼叫者开始传递信息，用XHR请求传输，通过EventSource推送到被呼叫者那。

WebRTC之所以不提供信令实现，是为了避免冗余，以及做到与现有技术的最大兼容，信令方法和协议都不由WebRTC标准来指定。

但WebRTC使用`JSEP协议`建立会话，采用`ICE`实现NAT穿越

> JSEP(JavaScript Session Establishment Protocol, JavaScript会话建立协议)是一个信令API，允许开发者构建更强大的应用程序以及增加在信令协议选择上的灵活性

WebRTC虽然没有指定具体的信令协议，但是媒体协商统一采用`SDP协议`。SDP格式含义可以参考[IETF examples](https://datatracker.ietf.org/doc/draft-nandakumar-rtcweb-sdp/?include_text=1)。`JSEP`一方面提供接口如createOffer()供web应用程序调用生成`SDP`，另一方面提供`ICE`功能接口。这些功能都由浏览器实现。以上用例就是直接使用JSEP生成offer/answer信令，然后采用ws协议传输实现的。

RTCPeerConnection接口被WebRTC应用用于创建各点之间的连接并交流音视频信息，在开始之前RTCPeerConnection需要做两个工作：

- 确定本地媒体情况，比如分辨率和编解码器的能力。这些元数据会用在offer/answer机制中
- 获取可能的应用主机网络地址，即candidate

当本地信息被确认后，就会通过信令系统与远程终端进行交换

在WebRTC交换网络信息，即finding candidates是通过`ICE框架`查找网络链接和端口的过程

## ICE应付NAT和防火墙

在真实网络环境中，大多数设备都处于一层或多层NAT之后，还有杀毒软件的阻挡一些端口或协议，或者使用了代理或防火墙。防火墙和NAT事实上可能都在同一个设备上，如家庭无线路由器

WebRTC使用ICE框架来克服实际应用中复杂的网络问题。要使用ICE，必须在RTCPeerConnection中传递ICE服务器的URL。ICE试图找到连接端点的最佳路径。并行的查找所有可能性，然后选择最有效的一项。ICE首先用从设备操作系统和网卡上获取的主机地址尝试连接，如果失败了(比如设备处于NAT之后)，ICE会使用从STUN服务器获取到的外部地址，如果仍然失败，再转交给TURN中继服务器来连接。

STUN服务器处于公网中并有个简单任务：检查请求(来自运行与NAT之后的应用)的IP: PORT地址，并且将这个地址响应回去。这个过程使得WebRTC终端可以找到自己公共访问方法，并通过信令机制将其发送给其他终端，就可以创建一个直连连接。

TURN其实就是RTCPeerConnection尝试用UDP协议建立终端间的直连失败后，改用TCP协议做终端间的数据中继。

每一个TURN服务器都支持STUN，因为TURN就是在STUN服务器中内建了一个中继功能。ICE也可以应付NAT复杂的设定。

## STUN

NAT对待UDP的实现方式有四种：

- Full Cone NAT：完全锥形NAT，所有从同一个内网IP和端口号发送过来的请求都会被映射成同一个外网IP和端口号，并且任何一个外网主机都可以通过这个映射的外网IP和端口号向这台内网主机发送包
- Restricted Cone NAT：限制锥形NAT，和Full Cone NAT基本流程一样，唯一不同的是，外网主机只能够向先前已经向它发送过数据包的内网主机发送包
- Port Restricted Cone NAT：端口限制锥形NAT，和Restricted Cone NAT基本一致，只不过包括端口号加入限制，只有内网主机先前已经给这个IP地址和端口发送过数据包
- Symmetric NAT：对称NAT，所有从同一个内网IP和端口号发送到一个特定的目的IP和端口号的请求，都会被映射到同一个IP和端口号。如果同一台主机使用相同的源地址和端口号发送包，但是发往不同的目的地，NAT将会使用不同的映射。此外，只有收到数据的外网主机才可以反过来向内网主机发送包。

### RFC3489/STUN

STUN(Simple Traversal of User Datagram Protocol Through Network Address Translators)，简单的用UDP穿透NAT，是基于UDP的完整的穿透NAT的解决方案。允许应用程序发现它们与公共互联网之间存在的NAT和防火墙及其他类型。也可以让应用程序确定NAT分配给它们的公网IP地址和端口号。默认端口号是3478.

STUN消息包含20个字节的消息头，包括16位的消息类型，16位的消息长度和128位的事务ID即标识符。

**消息类型**:

- 0x0001：捆绑请求
- 0x0101：捆绑响应
- 0x0111：捆绑错误响应
- 0x0002：共享私密请求
- 0x0102：共享私密响应
- 0x0112：共享私密错误响应

**消息长度**:

消息大小的字节数，不包括20字节的头部。

**事务ID**:

128的标识符，用于随机请求和响应，请求与其对应的所有响应具有相同的标识符。

消息头之后是0或多个属性，每个属性进行TLV编码，包括16位的属性类型、16位的属性长度和变长属性值。

**功能描述**: 客户通过带外方式获得STUN服务器信息后，就打开对应的地址和端口的连接，并开始与STUN服务器进行TLS协商。一旦打开了连接，客户就通过TCP协议发送共享私密请求，服务器生成共享私密响应。STUN在客户和服务器间使用共享私密，用作捆绑请求和捆绑响应中的密匙。之后，客户使用UDP协议向STUN服务器发送捆绑请求，当捆绑请求消息到达服务器的时候，它可能经过了一个或者多个NAT。结果是STUN服务器收到的捆绑请求消息的源IP地址被映射成最靠近STUN服务器的NAT的IP地址，STUN服务器把这个源IP地址和端口号复制到一个捆绑响应消息中，发送回拥有这个IP地址和端口号的客户端。

### RFC5389/STUN

在RFC5389中，重新命名为Session Traversal Utilities for NAT，即NAT会话穿透效用。

**用途**:

- ICE(Interactive Connectivity Establishment)，交互式连接建立
- SIP-OUTBOUND(Client-initiated connections for SIP)，SIP的客户端初始化连接
- BEHAVE-NAV(NAT Behavior Discovery)，NAT行为发现

STUN消息头为20字节，后面紧跟0或多个属性。STUN头部包含-STUN消息类型(16位)、magic cookie(32位)、事务ID(96位)和消息长度(16位)

### RFC5389与RFC3489的不同点

- 去掉STUN是一种完整的NAT穿透方案的概念，现在是一种用于提供NAT穿透解决方案的工具。因而，协议的名称变为NAT会话穿透效用；
- 定义了STUN的用途；
- 去掉了STUN关于NAT类型检测和绑定生命期发现的用法，去掉了RESPONSE-ADDRESS、CHANGED-ADDRESS、CHANGE-REQUEST、SOURCE-ADDRESS和REFLECTED-FROM属性；
- 增加了一个固定的32位的魔术字字段，事务ID字段减少了32位长度；
- 增加了XOR-MAPPED-ADDRESS属性，若魔术字在捆绑请求中出现时，该属性包括在捆绑响应中。否则，RFC3489中的行为是保留的（换句话说，捆绑响应中包括MAPPED-ADDRESS）；
- 介绍了消息类型字段的正式结构，带有一对明确的位来标识Request、Response、Error-Response或Indication消息。因此，消息类型字段被划分为类别和方法两部分；
- 明确的指出了STUN的最高2位是0b00，当用于ICE时可以简单的与RTP包区分开来；
- 增加指纹属性来提供一种明确的方法来检测当STUN协议多路复用时，STUN与其他协议之间的差异；
- 增加支持IPv6，IPv4客户端可以获取一个IPv6映射地址，反之亦然；
- 增加一个long-term-credential-based认证机制；
- 增加了SOFTWARE、REALM、NONCE和ALTERNATE-SERVER属性；
- 去掉了共享密匙方法，因此PASSWORD属性也去掉了；
- 去掉了使用连续10秒侦听STUN响应来识别一个攻击的做法；
- 改变事务计时器来增加TCP友好性；
- 去掉了STUN例子如集中分离控制和媒体面，代替的，在使用STUN协议时提供了更多的信息；
- 定义了一类填充机制来改变长度属性的说明；
- REALM、SERVER、原因语句和NONCE限制在127个字符，USERNAME限制在513个字节以内；
- 为TCP和TLS改变了DNS SRV规程，UDP仍然和以前保持一致。

[rfc5389](https://tools.ietf.org/html/rfc5389)

[rfc3489](https://tools.ietf.org/html/rfc3489)

## 部署STUN和TURN服务器

Google运行了一个公用的STUN服务器用作测试，`stun.l.google.com:19302`。

建议使用rfc766-turn-server当做产品用途的STUN/TURN服务，查看[源代码](https://code.google.com/archive/p/rfc5766-turn-server/)。

备选TURN服务器[restund](http://www.creytiv.com/restund.html)，在Google Compute Engine安装restund：

1. 防火墙开放tcp=443,udp/tcp=3478
2. 创建4个实例(?)，各自的公用IP，使用标准Ubuntu 12.06镜像
3. 配置本地防火墙(允许所有访问源)
4. 安装工具:
   `sudo apt-get install make`
   `sudo apt-get install gcc`
5. 从[creytiv.com/re.html](http://creytiv.com/re.html)安装libre
6. 从[creytiv.com/restund.html](http://creytiv.com/restund.html)获取restund并解包
7. wget [hancke.name/restund-auth.patch](http://hancke.name/restund-auth.patch) 并应用`patch -p1 < restund-auth.patch`
8. 对libre和restund运行 make, `sudo make install`
9. 按你自己的需求配置*restund.conf*(替换IP地址，确保正确的共享密钥)并复制到`/etc`目录
10. 复制*restund/etc/restund*到*/etc/init.d/*
11. 配置restund:
    设置*LD_LIBRARY_PATH*
    复制*restund.conf*到*/etc/restund.conf*
    设置*restund.conf*使用之前配的IP地址
12. 运行restund
13. 在远程机器运行stund client命令做测试: `./client IP:port`

## 超越端对端--多方WebRTC通讯

WebRTC应用可以使用多RTCPeerConnection，让各终端之间以网状配置连接。不过，CPU和带宽都消耗非常多，尤其是在移动终端上。

WebRTC应用可以按星状拓扑结构来选择一个终端分发数据流。在服务器运行一个WebRTC终端来作为重新分配机制也行

