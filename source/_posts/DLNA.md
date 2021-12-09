# HTTP协议

HTTP协议是基于TCP协议的应用层传输协议。

HTTP是一种无状态协议，HTTP本身不会对发送过的请求和相应的通信状态进行持久化处理。这样做的目的是为了保持HTTP协议的简单性，从而能够处理大量的事务，提高效率。

## TCP/IP协议族各层

### 应用层

FTP、DNS、HTTP等

### 传输层

传输层对上层应用层，提供处于网络连接中的两台计算机之间的数据传输。

#### TCP传输控制协议

#### UDP用户数据报协议


### 网络层

### 链路层




UPnP的端点的HTTP客户端不能报告支持HTTP/1.1 除非支持分段（thunked)传输。只有支持分段传输编码和100（继续响应）消息的客户端才支持HTTP/1.1

UPnP端点（设备和控制点）的HTTP客户端和服务器应支持持久性的HTTP/1.0 连接和流水线操作。

# DLNA 媒体传输协议
## 内容源和内容接收必须强制支持HTTP传输(必须)
## 内容源和内容接收可以支持RTP传输(可选)
## 传输模式
### 
传输模式的选择：内容接收器为接收的内容请求特定的传输模式。 或者在上传时内容源指定传输模式
## 在网速较慢的情况下允许与用户交互，提示用户是否需要停止渲染或者下载。
## 在流传输模式下运行的内容接收器能够以内容接收器支持的配置文件维持流传输所需的速率从网络接收内容
## 随机访问模型
### Content source如果支持随机访问，只能支持以下的一种。
#### 全随机访问模型

op-param与全随机访问模型绑定

如果内容源使用全随机访问模型，则需要满足以下条件：
- 整个二进制内容数据范围被定义为[S0, SN].
- S0数据边界应映射到固定的起点。

#### 有限随机访问模型

lop-npt,lop-bytes,lop-cleartextbytes与有限随机访问模型绑定

## HTTP传输
### HTTP 随机访问共通要求
HTTP Server端在响应GET请求中(数据传输中)，可以接受HEAD请求（用来获取当前可随机访问的数据范围）。
### HTTP 全随机访问
当使用全随机访问数据模型时
- npt 为0所对应的字节位置为0
- TimerSeekRange.dlna.org 报头所对应的字节位置对应解码器友好的字段。
#### Guideline
- 如果在具有Range或TimeSeekRange.dlna.org的GET请求中没有指定结束位置，那么HTTP Server端点的目标响应传输应包含当前可用的内容数据和当前流未来可用的内容数据。
- 如果HTTP服务器端点接收到忽略了Range和TimeSeekRange.dlna.org HTTP头的HTTP GET请求。那么HTTP服务器端点应推断出字节位置为0（从二进制内容的开始位置开始传输.
- 如果HTTP服务器端点支持全随机访问数据模型的范围HTTP头，那么HTTP服务器在HTTP GET/HEAD请求的目标响应中，应在Content-Range头部字段指明实例长度。
- 如果HTTP客户端想要获取内容二进制文件的实例长度，那么应该使用带有Range的HTTP请求头的HEAD请求（first-byte-position指定为0，并且忽略结束字节位置）（长度未知时响应*）。
- 如果HTTP服务器端点正在服务于存储的内容，那么它应在Content-Range头部字段中提供实例长度。
- 如果HTTP服务端点支持具有完全随机访问数据模型的TimeSeekRange.dlna.org HTTP头，则HTTP服务端点应在HTTP头部字段的TimerSeekRange.dlna.org内指定实例持续时间。

### HTTP 有限随机访问
#### Guideline
- 如果在具有Range或TimeSeekRange.dlna.org的GET请求中没有指定结束位置。那么HTTP服务端点的目标响应传输应包含当前可用的内容数据和未来对当前流将可用的内容数据。
> 对于活动内容，二进制内容的结尾没有定义。如果在请求中没有指定结束位置，那么意味着客户端希望继续接收数据，直到流的结束。
- 由于[R0,RN]的随机访问数据范围可以随时更改，因此HTTP客户端端点不应假定访问以前随机访问数据范围的GET请求始终可以成功（406 error response）。
- 如果HTTP客户端的GET请求中同时包含TimeSeekRange.dlna.org和PlaySpeed.dlna.org HTTP头部字段，则应该在TimeSeekRange.dlna.org的http请求头指定请求范围的结束位置。
> 即使HTTP服务器支持服务器端的trick-mode的PlaySpeed.dlna.org。在有限随机访问数据模型下，它也无法处理超过目前可用范围的数据。
- 如果服务端点支持有限访问模型的Range http header 或者 TimeSeekRange.dlna.org http header. 那么服务端点需要支持HTTP GET/HEAD 请求中的getAvailableSeekRange.dlna.org http header和availableSeekRange.dlna.org。
- 如果支持availableSeekRange.dlna.org的服务端点收到带有getAvaliableSeekRange.dlna.org http请求头的 GET/HEAD请求。那么服务端点将会回复当前可随机访问的数据范围。
- mode-flag = 1/0 分别表示S0 不可变和可变两种模式








