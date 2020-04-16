# Frontier & TcpServer

## 介绍

服务端的消息推送是通过长连接实现的，但是由于功能机不能接入`Frontier`的`sdk`，因此针对功能机实现一套基于`TCP`的长连接推送服务。

`Frotier`是公司的内部长连接通用组件，基于`WebSocket`实现，主要用于解决消息的实时推送，简要的架构设计参考文档 https://docs.google.com/document/d/1IDjl5u1lOEWHtgyHl4NY7hkhVllbKpHqohf0mpm4vms/edit.

`TcpServer`是基于`TCP`自建长连接应用层协议，主要用于与功能机的交互。

## 应用

这两套组件都是用于消息的实时推送，在项目中的具体使用如下：

![截屏2020-04-14下午8.26.02](/Users/bytedance/Desktop/截屏2020-04-14下午8.26.02.png)

1. 设备上线/下线时会在服务端维护设备的在线状态
2. 长连接用于消息的全双工上行与下行

## 实现

### 消息格式

不论是`WebSocket`还是`TCP`，两者的通信格式均为`Frame`结构，只不过两套方案的消息帧格式稍微有点区别。

#### Frontier

```go
message Frame {
	required uint64	seqid		= 1;
	required uint64	logid		= 2;

	required int32 service 	= 3;
	required int32 method		= 4;

	message ExtendedEntry {
		required string key   = 1;
		required string value = 2;
	}

	repeated ExtendedEntry headers	= 5; 

	optional string	payload_encoding= 6;  
	optional string	payload_type	= 7;  
	optional bytes	payload		= 8;  
}

```

1. seqid：主要可以用来进行消息除重的功能
2. service：frontier协议层通过该字段和appid来路由收到的frame，先查本地的路由映射规则表，然后将调用对应的service上的rpc。
3. method: BackService 接口标识，如 SendMessage/Auth/SendEvent
4. payload_type: 业务包的编码格式，如json/pb
5. payload：在指定 playload_type 编码格式下数据包的字节

在`Frame`中的`Service`字段即为我们的业务服务的`PSM`，需要在`Frontier`平台进行注册，并且实现相应的接口

* Auth: 长连接鉴权接口，用于在连接建立时进行用户鉴权；如果`BackService`中没有实现该接口，则默认使用头条内部的鉴权接口。在我们的业务中，`im_go_api`中实现了该接口。
* SendMessage: 消息上行接口，当客户端发送消息时，会将消息传送到`BackService`中的`SendMessage`接口，以便进行相应的业务处理。
* SendEvent: 长连接建立/断开的事件通知，在`UserStatus`中实现了该接口。

#### TcpServer

```go
type Frame struct {
	Cmd     byte
	SeqID   uint16
	Payload []byte
	length  int
}
```

1. Cmd: 用于标识服务端与客户端进行交互时的消息，可以取值如下：

```go
  CmdSysNull   = 0x00 // Internal use
	CmdSysHello  = 0x01 // Internal use
	CmdSysHelloX = 0x07 // Internal use
	CmdSysPing   = 0x02 // 维持心跳
	CmdSysAck    = 0x03 // Internal use
	CmdSysProxy  = 0x0E // Internal use
	CmdBye       = 0x0F // 连接断开
	CmdPush      = 0x08 // 消息推送
	CmdData      = 0x0A // 业务交互
```

2. Playload: 业务数据字节流。
3. Length：标识数据长度，用于拆包处理。

#### 区别

1. 由于`Frontier`是建立在`WebSocket`之上的，因此不需要额外进行拆包处理；而`TcpServer`建立在`Tcp`之上，数据流相对来说较为原始，需要手动拆包。
2. `Frontier`数据包可以指定多种编码格式，扩展性较高；`TcpServer`中与客户端约定编码格式固定为`ProtoBuf`

> 为了节省功能机的流量，`TcpServer`的传输包尽可能小

### 连接建立与维持

两者在长连接的建立与维持方面有些许不同，首先`Frontier`需要进行协议转换完成`WebSocket`的握手，而`TcpServer`只需要建立`Tcp`的长连接即可；其次，`Frontier`在连接建立之前就需要进行用户鉴权，`TcpServer`是在连接建立之后 业务请求时进行用户鉴权；最后，`Frontier`的心跳维持是服务端主动`Ping`，客户端可选`Ping`，而`TcpServer`是客户端进行`Ping`。

#### 连接结构体

```go
type Connection struct {
	UUID uuid.UUID

	UserID    int64
	DeviceID  int64
	...
	Querystring string
	IPAddr      string
	Port        string

	closed bool

	// The websocket connection.
	conn net.Conn
	...

	pingInterval int32
	...
	st ConnStat
}

// uuid
type UUID struct {
	t    uint32
	cnt  uint32
	ip   [net.IPv4len]byte
	port uint16
}
```

1.  `uuid` 用于唯一标识一个长连接，包含一个依赖当前时间戳生成的32位标识，当前主机的 `ip`，`port`及一个原子自增变量

> 在Frontier的优化过程中，之前是采用 userId + deviceId 标识长连接，但是不够唯一性，容易被伪造并且存在旧连接把新连接覆盖掉

2. `conn`代表`websocket`的长连接，用于数据的读写

> 在 Frontier 文档介绍中，对 web socket 进行了优化：1. 为了减少内存占用，删除了原本自带的buffer，并实现了零拷贝；2. 由于 Frotier 采用的是服务端主动ping，实现方式是利用 read timeout 之后进行 ping，因此是对原本的 read 操作进行了重写。

> 在 Frontier 的优化文档中提出，为了减小长连接内存占用，降低了长连接的收发缓存；看了一些资料，tcp 的收发缓存虽然占据一定的空间，但是只有在读写处理能力不足时才会分配相应的缓存空间，而我们的TcpServer基本是建立在原生的 tcp 之上，应该不存在缓存优化的问题。

3. `pingInterval`服务端主动发送心跳的时间间隔

> 长连接的读超时

4. `st`长连接的状态，分为`SendPing`,`ReadMessgae`,`WriteMessage`等状态

长连接信息会在连接建立之后注册到`BackBone`中，用于后续的消息推送。



```go
type connState struct {
	*evio.Conn
	rd     blockedReader  // Pending bytes for reading
	authed bool           // Auth state
	key    []byte         // Key, if presented, will be used to encrypt/decrypt payload
	dieKey sched.SchedKey // Timeout handler
	// buf       []byte
}
```

1. `evio.Conn`：代表`tcp`长连接，用于读写数据；相对于`WebSocket`的长连接更为底层，可控性更强。
2. `authed`：标识是否鉴权成功
3. `key`：密钥，暂未使用
4. `dieKey`：记录长连接的存活时间，超过指定的时间未收到客户端的`ping`，服务端会主动断开长连接。

#### 连接建立流程

![截屏2020-04-15下午4.39.00](/Users/bytedance/Desktop/截屏2020-04-15下午4.39.00.png)

1.  客户端发起`http`请求进行协议转换，转换之前需要进行用户鉴权，如果指定了`BackService.Auth`接口，则调用自定义鉴权功能，否则使用默认的鉴权系统；鉴权成功之后进行协议转换并建立`WebSocket`长连接。
2. 服务端生成长连接标识（`Connection`），包含了`uuid`,`ip`,`port`,`userId`,`deviceId`

> userId 由 Auth 接口返回

3. 将长连接标识注册到`BackBone`，同时触发上线事件`SendEvent`，而我们收到上线事件之后将用户的在线状态注册到`UserStatus`中

> `SendEvent`默认重试三次；BackBone /UserStatus 将用户信息保存到 Redis 。



![截屏2020-04-15下午5.48.27](/Users/bytedance/Desktop/截屏2020-04-15下午5.48.27.png)

1. 功能机发起`tcp`建立请求，当服务端成功建连之后，维持心跳计时（默认为 20 分钟）

> 如果开启了keep-alive，默认时间为 10 分钟

2. 服务端本地维持 conid —> connection 的映射

> Conid 是在 socket fd 基础上进行生成的

3. 在功能机通过长连接发送请求/心跳包时，会将连接信息注册到`UserStatus`中

> 即使是心跳包，也会在每次 ping 的时候进行注册---> 是不是可以忽略这个动作

#### 心跳维护

`Frontier`在维护心跳时，是由服务端主动发送心跳包；心跳是为了在较长一段时间内没有数据传输时，维护长连接活性，`Frontier`通过在读取超时时由服务端发送心跳包，这样就不用单独为每个长连接维护一个定时策略，减少cpu的消耗。

```go
if ok && ne.Timeout() && c.wsr.Reentrant() && time.Since(c.GetAccessTime()) < c.PingInterval(2) {
  // send ping
}
```

`TcpServer`采用端上定时发送心跳包的策略，服务端会为每个长连接维护一个定时机制，即`dieKey sched.SchedKey`字段。该计时器会在有数据传输时重新计时，超时之后会主动清除连接信息，断开连接。

### 用户在线状态维护

长连接建立/断开会修改用户的在线状态，`UserStatus`负责存储用户当前是否在线，在线设备等信息，对外暴露两种接口`SendEvent`（智能机），`Register`/`UnRegister`(功能机)。

维护的信息包括：

```go
type UserClient struct {
	DeviceId     string       
	ClientAddr   string       // 客户端IP
	ClientType   ClientType   // 客户端类型：功能机；智能机
	ConnServer   string       // 长连接服务端地址
	ConnId       int64        // 功能机的connid
	Ts           int64        
	OnlineStatus OnlineStatus 
	ConnUuid     string       // 智能机的connid
}
```

目前是将智能机与功能机的状态维护两套（redis key 不同）

> 之前是将两种设备状态维护一套，会存在过期信息覆盖的情况：智能机下线之后长连接可能不会立马断开，所以如果是功能机把智能机踢下线，那么设备的在线状态会变成：智能机在线---> 功能机在线---> 智能机离线

### 消息上行与下行

#### 消息上行

![截屏2020-04-15下午7.51.38](/Users/bytedance/Desktop/截屏2020-04-15下午7.51.38.png)

针对`Frontier`存在`play_load`字段，用于存储编码之后的消息结构体。`Frontier`长连接监听数据传输，如果是业务数据，则会调用对应的`BackService`发送消息。以RA为例，`im_go_api`在接收到信息之后进行参数校验，之后提取`paly_load`中的数据，解析其中的`CMD`，如`SendMessage`/`CreateConversation`，然后找到对应的处理函数。

> `im_go_api`是以CMD进行接口处理，意思是如果短连的url与CMD不一致的话，会以CMD为准。

`TcpServer`的消息上行即为`Http`请求。

#### 消息下行

![截屏2020-04-15下午8.02.16](/Users/bytedance/Desktop/截屏2020-04-15下午8.02.16.png)

我们的消息推送都需要经过`MessageConsuemr`进行处理，所以当有消息需要进行推送时`MessageConsumer`会先查询`userStatus`判断当前用户的在线设备及在线状态。

> 如果当前是功能机离线，则不会进行推送处理；如果是智能机离线，则会使用pns推送

在进行消息推送的时候，两者采用的思路基本一致：根据`connid`确定长连接信息（`ip`,`port`）找到长连接服务所在的机器，然后调用`Frontier`/`TcpServer`进行长连接推送。

```go
func GetFrontierClient(addr string) *frontier.FrontierClient {
	frontierMu.RLock()
	cli := frontierClients[addr]
	frontierMu.RUnlock()
	if cli != nil {
		return cli
	}

	frontierMu.Lock()
	defer frontierMu.Unlock()
	cli = frontierClients[addr]
	if cli != nil {
		return cli
	}
	u := &url.URL{Scheme: "tcp", Host: addr, RawQuery: "timeout=100ms"}
	cli = frontier.NewFrontierClient(gauss.NewClient(u, gauss.WithMaxIdleConn(10)))
	frontierClients[addr] = cli
	return cli
}
```

```go
instances, err := discovery.ConsulDiscover(tcpClient.Name(), env.IDC())
	if err != nil || len(instances) == 0 {
		logs.CtxError(ctx, "[Push] no tcp instance online")
		return nil, errors.New("no tcp instance online")
	}

	index := -1
	for i, instance := range instances {
		if instance.Host == addr {
			index = i
			break
		}
	}
```

