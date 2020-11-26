# RTMP 协议

转自[RTMP 协议](https://www.jianshu.com/p/d511d59b185c)

## 一、概述

<font size=3 color=666666>RTMP协议是Real Time Message Protocol(实时信息传输协议)的缩写，它是由Adobe公司提出的一种应用层的协议，用来解决多媒体数据传输流的多路复用（Multiplexing）和分包（packetizing）的问题。</font>



```undefined
握手、消息块概念
握手的目的是为了确认对端RTMP的Version和确认对端能互相通信。
消息块就是消息的载体，是RTMP协议最重要的载体，这个载体是有一定格式的，如果把Client和Server端当作铁路的两个站点，
那这个消息块就是火车，它负责运输货物。正如火车有火车头、车厢一样，消息块也有基本头，消息头和消息负载。RTMP协议当中，
除了握手协议，其他的数据都是以消息块的方式发送的，发送一个消息时，当块大小比需要发送的消息的字节数更大时，
一个消息块就相当于一个消息，否则消息需要分成多个消息块。
```

## 二、握手

<font size=3 color=666666>要建立一个有效的RTMP Connection链接，首先要“握手”:客户端要向服务器发送C0,C1,C2（按序）三个chunk，服务器向客户端发送S0,S1,S2（按序）三个chunk，然后才能进行有效的信息传输。RTMP协议本身并没有规定这6个Message的具体传输顺序，但RTMP协议的实现者需要保证这几点如下：</font>

- 客户端要等收到S1之后才能发送C2
- 客户端要等收到S2之后才能发送其他信息（控制信息和真实音视频等数据）
- 服务端要等到收到C0之后发送S1
- 服务端必须等到收到C1之后才能发送S2
- 服务端必须等到收到C2之后才能发送其他信息（控制信息和真实音视频等数据）

### 握手流程

握手是一切的开始，Client和Server两个站点之前要运输货物，首先得先互相通知对方，确认铁轨是否符合火车运行，是否畅通无阻，是否能准确的运输货物到对端。
 实际的流程大概是这样的：

1. Client发送带有1byte的C0和固定长度为1536byte的C1。
2. Server发送S0S1S2给Client。
3. Client发送C2。

```ruby
    +-+-+-+-+-+-+            +-+-+-+-+-+-+
    |  Client   |            |  Server   |
    +-+-+-+-+-+-+            +-+-+-+-+-+-+
          |--------- C0C1 -------->|     
          |<------- S0S1S2  -------|
          |---------- C2 --------->|
```

- C0和S0的格式（1 byte）

```ruby
    +-+-+-+-+-+-+-+-+
    |  version      |
    +-+-+-+-+-+-+-+-+
```

- C1和S1的格式

```ruby
    +-+-+-+-+-+-+-+-+-+-+
    |   time (4 bytes)  |
    +-+-+-+-+-+-+-+-+-+-+
    |   zero (4 bytes)  |
    +-+-+-+-+-+-+-+-+-+-+
    |   random bytes    |
    +-+-+-+-+-+-+-+-+-+-+
    |random bytes(cont) |
    |       ....        |
    +-+-+-+-+-+-+-+-+-+-+
```

```undefined
time: 4 字节
本字段包含时间戳。该时间戳应该是发送这个数据块的端点的后续块的时间起始点。
可以是 0 ,或其他的任何值。为了同步多个流,端点可能发送其块流的当前值。

zero: 4 字节
本字段必须是全零。

random bytes: 1528 字节。
本字段可以包含任何值。因为每个端点必须用自己初始化的握手和对端初始化的
```

- C2和S2的格式

```ruby
    +-+-+-+-+-+-+-+-+-+-+
    |   time (4 bytes)  |
    +-+-+-+-+-+-+-+-+-+-+
    |   time2(4 bytes)  |
    +-+-+-+-+-+-+-+-+-+-+
    |   random bytes    |
    +-+-+-+-+-+-+-+-+-+-+
    |random bytes(cont) |
    |       ....        |
    +-+-+-+-+-+-+-+-+-+-+
```

```ruby

time: 4 字节
本字段必须包含对等段发送的时间(对C2来说是S1 ,对S2来说是C1)。

time2: 4 字节
本字段必须包含先前发送的并被对端读取的包的时间戳。

random bytes: 1528 字节
本字段必须包含对端发送的随机数据字段(对C2来说是S1 ,对S2来说是C1)。  
```

## 三、消息块 Chunck Block

<font size=3 color=666666>RTMP在收发数据的时候并不是以Message为单位的，而是把Message拆分成Chunk发送，而且必须在一个Chunk发送完成之后才能开始发送下一个Chunk。每个Chunk中带有MessageID代表属于哪个Message，接受端也会按照这个id来将chunk组装成Message。</font>

> 为什么RTMP要将Message拆分成不同的Chunk呢？通过拆分，数据量较大的Message可以被拆分成较小的“Message”，这样就可以避免优先级低的消息持续发送阻塞优先级高的数据，比如在视频的传输过程中，会包括视频帧，音频帧和RTMP控制信息，如果持续发送音频数据或者控制数据的话可能就会造成视频帧的阻塞，然后就会造成看视频时最烦人的卡顿现象。同时对于数据量较小的Message，可以通过对Chunk Header的字段来压缩信息，从而减少信息的传输量。

每个Chunk都由 <font size=3>**Chunk Header + Chunk Data**</font> 组成

```ruby
  +-------+     +--------------+----------------+  
  | Chunk |  =  | Chunk Header |   Chunk Data   |  
  +-------+     +--------------+----------------+
```

### 3.1. Chunk Header 由 Basic Header + Message Header + ExtendedTimestamp(不一定存在)组成

```ruby
  +--------------+     +-------------+----------------+-------------------+ 
  | Chunk Header |  =  | Basic header| Message Header |Extended Timestamp |  
  +--------------+     +-------------+----------------+-------------------+
```

### 3.1.1. `Basic Header(基本的头信息)` （1~3 byte）

```ruby
  +-+-+-+-+-+-+-+-+
  |fmt|   cs id   |
  +-+-+-+-+-+-+-+-+
  chuck stream = cs
      
fmt： 表示块类型，决定了Chunk Msg Header的格式，它占第一个字节的0~1bit，
csid：表示块流id 占2~7bit属于csid。
　①csid在64~319的范围内时，
　　　csid =（第二个字节的值）+64
　　　 0                   1            
　　　 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
     |fmt|0 0 0 0 0 0|the second byte|
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
     
　②csid在64~65599的范围内时(①与②存在交集，原则上交集部分选择①)，
　　　bit位全为1时，csid=（第二个字节的值×256）+（第三个字节的值）+64
　　　 0                   1                   2       
　　　 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 
　　　+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
     |fmt|1 1 1 1 1 1|the second byte| the third byte|
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
     
　③csid在3~63的范围内时，
　　　即6位bit非全0也非全1时
　　　 0               
　　　 0 1 2 3 4 5 6 7 
　　　+-+-+-+-+-+-+-+-+
     |fmt|0<csid<0x3f|
     +-+-+-+-+-+-+-+-+
  * csid 0,1,2 保留
   如果读取csid值，首先读取前两位，即fmt值。然后读取后六位，如果全是0，则读取8位csid，如果全是1，就读取16位，如果不是全0或者全1，就记该值为csid
```

```objectivec
以下是上述 chuck stream id 类型非全部：
typedef NS_ENUM(NSUInteger, RTMPChunckStreamID)
{
   RTMPChunckStreamID_PRO_CONTROL       = 0x2, // 协议控制块流ID  
   RTMPChunckStreamID_COMMAND           = 0x3, // 控制块流ID  
   RTMPChunckStreamID_MEDIA             = 0x4, // 音视频块流ID  
};
```



**由此可见：**Basic Header的长度范围为1~3 byte，csid的值范围3～65599,0~2作为保留。

### 3.1.2. `Chuck Message Header（块消息的消息头信息）`（0、3、7、11 byte）

包含了要发送的实际信息（可能是完整的，也可能是一部分）的描述信息。Message Header的格式和长度取决于Basic Header的chunk type，共有4种不同的格式，由上面所提到的Basic Header中的fmt字段控制。
 其中第一种格式可以表示其他三种表示的所有数据，但由于其他三种格式是基于对之前chunk的差量化的表示，因此可以更简洁地表示相同的数据，实际使用的时候还是应该采用尽量少的字节表示相同意义的数据。以下按照字节数从多到少的顺序分别介绍这4种格式的Chuck Message Header

```go
   0                   1                   2                   3   
   0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 
  +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+  
  |                     timestamp                 | message length:           
  +-------------------------------+---------------+---------------+  
  :                               |message type id|               : 
  +-------------------------------+---------------+---------------+ 
  :                  message stream id            |  
  +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
  
  timestamp（时间戳）: 占 3 byte 最大表示16777215=0xFFFFFF=2^24-1,超出这个值，这3个字节置为1，将实际数据转存到Extended Timestamp字段中。
           
  message length（消息长度）: 占 3 byte 表示实际发送的消息的数据如音频帧、视频帧等数据的长度，单位是字节。注意这里是Message的长度，也就是chunk属于的Message的总数据长度，而不是chunk本身Data的数据的长度。
           
  message type id（消息的类型id）: 占 1 byte 表示实际发送的数据的类型，如8代表音频数据、9代表视频数据。
  
  message stream id（消息的流id）: 占 4byte 表示该chunk所在的流的ID，和Basic Header的CSID一样，它采用小端存储的方式。一个消息会按照chunk分发，带相同message stream id的chunk表示同一个message
  
 
  ①fmt=0：长度11 byte，其他三种能表示的数据它都能表示
    在一个块流的开始和时间戳返回的时候必须有这种块
    Chuck Message Header = timestamp + message length + message type id + message stream id
    
  ②fmt=1：长度为7 byte，与fmt=0时，相比，该类型少了Message Stream Id，
    具有可变大小消息的流,在第一个消息之后的每个消息的第一个块应该使用这个格式
    Chuck Message Header = timestamp + message length + message type id
    
  ③fmt=2：长度3 byte，不包含Message Stream Id和Message Length 、Message type Id
    具有固定大小消息的流,在第一个消息之后的每个消息的第一个块应该使用这个格式
    Chuck Message Header = timestamp
       
  ④fmt=3：长度为0 byte， 
    当一个消息被分成多个块,除了第一块以外,所有的块都应使用这种类型
    Chuck Message Header =  0 byte
  
```

```objectivec
以下是上述Basic Header中fmt值枚举：
typedef NS_ENUM(NSUInteger, RTMPBHFmt)
{
    RTMPBHFmt_FULL              = 0x0,
    RTMPBHFmt_NO_MSG_STREAM_ID  = 0x1,
    RTMPBHFmt_TIMESTAMP         = 0x2, // 'Chuck Message Header' only timestamp
    RTMPBHFmt_ONLY              = 0x3, // 'Chunk Message Header' all no
};

以下是上述 message type id 类型非全部:
typedef NS_ENUM(NSUInteger, RTMPMessageTypeID)
{
    RTMPMessageTypeID_CHUNK_SIZE     = 0x1, //协议控制消息 ChunkData承载大小，进行分块
    RTMPMessageTypeID_ABORT          = 0x2, //协议控制消息 消息分块只收到部分时，发送此控制消息，发端不在
    RTMPMessageTypeID_BYTES_READ     = 0x3, //协议控制消息
    RTMPMessageTypeID_PING           = 0x4, //用户控制消息 该消息在Chunk流中发送时，msg stream id = 0, chunck stream id = 2, message type id = 4
    RTMPMessageTypeID_SERVER_WINDOW  = 0x5, //协议控制消息
    RTMPMessageTypeID_PEER_BW        = 0x6, //协议控制消息
    RTMPMessageTypeID_AUDIO          = 0x8, //音频消息
    RTMPMessageTypeID_VIDEO          = 0x9, //视频消息
    RTMPMessageTypeID_FLEX_STREAM    = 0xF,
    RTMPMessageTypeID_FLEX_OBJECT    = 0x10,
    RTMPMessageTypeID_FLEX_MESSAGE   = 0x11,
    RTMPMessageTypeID_NOTIFY         = 0x12, //数据消息，传递一些元数据
    RTMPMessageTypeID_SHARED_OBJ     = 0x13, //
    RTMPMessageTypeID_INVOKE         = 0x14, //命令消息，客户端与服务器之间执行命令如：connect、publish
    RTMPMessageTypeID_METADATA       = 0x16, //
};
```

> <font size=4 color=ff0000>注意：</font>
>
> - message type id  发送音视频数据的时候
> - 如果包头MessageTypeID为0x8或0x9，数据(chunk data)是flv的tag data(没有tag header),flv格式封装请见官网
> - 也可以用新类型MessageTypeID为0x16，数据(chunk data)是一个完整flv的tag(tag header + tag data)
> - message stream id 采用<font size=4 color=000000>小端</font>存储
> - RTMP都是<font size=4 color=000000>大端</font>模式，所以发送数据，包头，交互消息都要填写<font size=4 color=000000>大端</font>模式的，但是只有streamID是<font size=4 color=000000>小端</font>模式



 <font size=4 color=000000>**附:**大小端的理解</font>

```ruby
例子:

假设某段内存中存放以下这样的数据
低地址                           高地址
------------------------------------>
+--------+--------+--------+--------+
|   11   |   66   |    85  |   27   |
+--------+--------+--------+--------+

大端模式下值为 0x11668527，数据高位存放在低地址中，数据低位存放高地址中
小端模式下值为 0x27856611，数据高位存放在高地址中，数据低位存放低地址中
所以在对数据处理的时候需要注意当前是在什么模式下，一般32位环境是小端模式，64位环境是大端模式
```

### 3.1.3. `ExtendedTimestamp（扩展时间）` （0、4 byte）

```ruby
   0                   1                   2                   3   
   0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 
  +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+ 
  |                           timestamp                           |   
  +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
  
  只有当块消息头中的普通时间戳设置为 0xffffff 时,本字段才被传送。  
  如果普通时间戳的值小于 0x00ffffff ,那么本字段一定不能出现。
  如果块消息头中时间戳字段不出现本字段也一定不能出现。
  类型 3 的块一定不能含有本字段。
  本字段在块消息头之后,块数据之前。 
```

### 3.2.Chunk Data

```ruby
  +-----------+
  |Chunk Data |
  +-----------+
```

```undefined
Chunk Data的实例就是Message
```

### 3.2.1.消息-Message

### 3.2.2.消息的分类

#### 3.2.2.1

#### 协议控制消息Message Type ID 的范围1，2，3，5，6

> 协议控制消息是用来与对端协调控制的
>
>  1，2 用于chunk协议
>
>  3，4，5，6 用于rtmp协议本身，协议控制消息必须要求
>
>  **Message Stream ID=0 和 Chunk Stream ID=2**
>   
>  这个是立即生效的，会忽略timestamps

- <font size=3 color=#666666 >**MT=1**, Set Chunk Size 设置块的大小，通知对端用使用新的块大小,共4 bytes，但是第一位必须是0，31位可用。默认大小是128字节</font>

```ruby
  +-------------+----------------+-------------------+----------------+  
  | Basic header|Chunk Msg Header|Extended Timestamp | Set chunk size |  
  +-------------+----------------+-------------------+----------------+ 

0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|0| chunk size (31 bits)                                        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

```

<font size=3 color=#666666 >**MT=2**, Abort Message 取消消息，用于通知正在等待接收块以完成消息的对等端,丢弃一个块流中已经接收的部分并且取消对该消息的处理，共4 bytes。</font>

```ruby
  +-------------+----------------+-------------------+----------------+  
  | Basic header|Chunk Msg Header|Extended Timestamp | Chunk Stream ID|  
  +-------------+----------------+-------------------+----------------+ 
```

<font size=3 color=#666666 >**MT=3**, Acknowledgement 确认消息，客户端或服务端在接收到数量与窗口大小相等的字节后发送确认消息到对方。窗口大小是在没有接收到接收者发送的确认消息之前发送的字节数的最大值。服务端在建立连接之后发送窗口大小。本消息指定序列号。序列号,是到当前时间为止已经接收到的字节数。
 共4 bytes。</font>

```ruby
  +-------------+----------------+-------------------+----------------+  
  | Basic header|Chunk Msg Header|Extended Timestamp | Sequence Number|  
  +-------------+----------------+-------------------+----------------+ 
```

<font size=3 color=#666666 >**MT=5**, Window Acknowledgement Size 确认窗口大小,客户端或服务端发送本消息来通知对方发送确认消息的窗口大小,共4 bytes.</font>

```ruby
  +-------------+----------------+-------------------+----------------------------+  
  | Basic header|Chunk Msg Header|Extended Timestamp | Window Acknowledgement Size|  
  +-------------+----------------+-------------------+----------------------------+ 
```

<font size=3 color=#666666 >**MT=6**, Set Peer Bandwidth 设置对等端带宽，客户端或服务端发送本消息更新对等端的输出带宽。发送者可以在限制类型字段（1 bytes）把消息标记为硬(0),软(1),或者动态(2)。如果是硬限制对等端必须按提供的带宽发送数据。如果是软限制,对等端可以灵活决定带宽,发送端可以限制带宽?。如果是动态限制,带宽既可以是硬限制也可以是软限制。</font>

```dart
+-------------+---------------+------------------+---------------------------+-----------+ 
|Basic header|Chunk Msg Header|Extended Timestamp|Window Acknowledgement Size|Limit type |
+------------+----------------+------------------+---------------------------+-----------+ 
  
  Hard(Limit Type＝0):接受端应该将Window Ack Size设置为消息中的值
  Soft(Limit Type=1):接受端可以讲Window Ack Size设为消息中的值，也可以保存原来的值（前提是原来的Size小与该控制消息中的Window Ack Size）
  Dynamic(Limit Type=2):如果上次的Set Peer Bandwidth消息中的Limit Type为0，本次也按Hard处理，否则忽略本消息，不去设置Window Ack Size。 
```




## User Control Messages 

<font size=3 color=#666666 >**MT=4**, User Control Message 用户控制消息，客户端或服务端发送本消息通知对方用户的控制事件。本消息承载事件类型和事件数据。消息数据的头两个字节用于标识事件类型。事件类型之后是事件数据。事件数据字段是可变长的。</font>

```ruby
  +-------------+----------------+-------------------+-----------+-----------+  
  | Basic header|Chunk Msg Header|Extended Timestamp | Event Type|Event Datar|  
  +-------------+----------------+-------------------+-----------+-----------+ 
```



#### 3.2.2.2.音频数据消息

- <font size=3 color=#666666 >**MT=8**, Audio message, 客户端或服务端发送本消息用于发送音频数据。</font>

#### 3.2.2.3.视频数据消息

- <font size=3 color=#666666 >**MT=9**, Video message, 客户端或服务端使用本消息向对方发送视频数据。</font>

#### 3.2.2.4.元数据消息

- <font size=3 color=#666666 >**MT=15或18**,  Data message, 客户端或服务端通过本消息向对方发送元数据和用户数据。元数据包括数据的创建时间、时长、主题等细节。
   消息类型为 18 的用 AMF0 编码,消息类型为 15 的用AMF3 编码。</font>

#### 3.2.2.5.共享对象消息

- <font size=3 color=#666666 >**MT=16或19**,共享库是一个Flash对象（名称值对的集合），它们在多个客户端，实例等之间同步。 AMF0的消息类型19和AMF3的消息类型16保留用于共享对象事件。 每条消息可以包含多个事件。</font>

  The shared object message format

```
+------+------+-------+-----+-----+------+-----+ +-----+------+-----+
|Header|Shared|Current|Flags|Event|Event |Event|.|Event|Event |Event|
|	   |Object|Version|	    |Type |data  |data |.|Type |data  |data |
|	   |Name  |		  |		|	  |length|	   |.| 	   |length|		|
+------+------+-------+-----+-----+------+-----+ +-----+------+-----+
	   |															|
	   |<- - - - - - - - - - - - - - - - - - - - - - - - - - - - - >|
	   |		AMF Shared Object Message body 						|
The shared object message format
```

 event types are supported:

```
+---------------+--------------------------------------------------+
| Even			|Description										|
+---------------+--------------------------------------------------+
| Use(=1)		| The client sends this event to inform the server  
|		 		| about the creation of a named shared object.		|
+---------------+--------------------------------------------------+
| Release(=2)	| The client sends this event to the server when	|
|				| the shared object is deleted on the client side. |
+---------------+--------------------------------------------------+
| Request Change| The client sends this event to request that the  
| (=3)			| change the value associated with a named			|
| 				| parameter of the shared object.					|
+---------------+--------------------------------------------------+
| Change (=4)	| The server sends this event to notify all 		|
|				| clients, except the client originating the		|
|				| request, of a change in the value of a named		|
|				| parameter											|
+---------------+--------------------------------------------------+
| Success (=5) 	| The server sends this event to the requesting  	|
| 				| client in response to RequestChange event if the  
|				| request is accepted. 								|	
+---------------+--------------------------------------------------+
| SendMessage	| The client sends this event to the server to		|	
| (=6)			| broadcast a message. On receiving this event,		|
|				| the server broadcasts a message to all the		|
|				| clients, including the sender.					|
+---------------+--------------------------------------------------+
| Status (=7)	| The server sends this event to notify clients		|
|		 		| about error conditions.							|
+---------------+--------------------------------------------------+
| Clear (=8)	| The server sends this event to the client to		|
|				| clear a shared object. The server also sends		|
|				| this event in response to Use event that the		|
|				| client sends on connect.							|
+---------------+--------------------------------------------------+
| Remove (=9)	| The server sends this event to have the client	|
|				| delete a slot.									|
+---------------+--------------------------------------------------+
| Request Remove| The client sends this event to have the client	|
| (=10)			| delete a slot.									|
+---------------+--------------------------------------------------+
| Use Success	| The server sends this event to the client on a	|
| (=11)			| successful connection.							|
+---------------+--------------------------------------------------+

```



#### 3.2.2.6.命令消息

客户端和服务器交换被AMF编码的命令。 发送方发送一条命令消息，该消息由命令名称，事务ID和包含相关参数的命令对象组成。 例如，connect命令包含“ app”参数，该参数告诉客户端连接到的服务器应用程序名称。 接收方处理该命令，并发送回具有相同事务ID的响应。 响应字符串可以是`_result`，`_error`或方法名称，例如verifyClient或contactExternalServer。

`_result`或`_error`的命令字符串表示响应。 事务ID指示响应所引用的未完成命令。 它与IMAP和许多其他协议中的标记相同。 命令字符串中的方法名称表示发送方正在尝试在接收方运行方法。

以下类对象用于发送各种命令：
NetConnection一个对象，是服务器和客户端之间连接的高级表示。
NetStream一个对象，表示通过其发送音频流，视频流和其他相关数据的通道。 我们还发送诸如播放，暂停等命令，这些命令控制数据流。



NetConnection管理客户端应用程序和服务器之间的双向连接。 此外，它还支持异步远程方法调用。

NetConnection 有	 connect call close  createStream

命令的编码形式有AMF0和AMF3,

```
+----------------------+----------------------------+--------------+
|	 Encoding Type	   |         	Usage 			|	  Value		|
+----------------------+----------------------------+--------------+
|		AMF0		   |    AMF0 object encoding	|		0		|
|					   | supported by Flash 6 and	|				|
|					   |            later			|				|
+----------------------+----------------------------+--------------+
|		AMF3		   |     AMF3 encoding from		|		3		|
|					   |  		 Flash 9 (AS3)		|				|
+----------------------+----------------------------+--------------+
```



- <font size=3 color=#666666 >**MT=17或20**, Command message, 命令消息都是用AMF编码的，AMF有两种，为AMF0和AMF3。命令消息有命令名，传输ID，和命名对象组成。而命名对象是由一系列参数组成的。
   </font>

```c++
// The amf0 command message, command name macros
// amf0命令如下
#define RTMP_AMF0_COMMAND_CONNECT               "connect"
#define RTMP_AMF0_COMMAND_CREATE_STREAM         "createStream"
#define RTMP_AMF0_COMMAND_CLOSE_STREAM          "closeStream"
#define RTMP_AMF0_COMMAND_PLAY                  "play"
#define RTMP_AMF0_COMMAND_PAUSE                 "pause"
#define RTMP_AMF0_COMMAND_ON_BW_DONE            "onBWDone"
#define RTMP_AMF0_COMMAND_ON_STATUS             "onStatus"
#define RTMP_AMF0_COMMAND_RESULT                "_result"
#define RTMP_AMF0_COMMAND_ERROR                 "_error"
#define RTMP_AMF0_COMMAND_RELEASE_STREAM        "releaseStream"
#define RTMP_AMF0_COMMAND_FC_PUBLISH            "FCPublish"
#define RTMP_AMF0_COMMAND_UNPUBLISH             "FCUnpublish"
#define RTMP_AMF0_COMMAND_PUBLISH               "publish"
#define RTMP_AMF0_DATA_SAMPLE_ACCESS            "|RtmpSampleAccess"
```





**命令消息的类型**

命令基本是下面格式。每个命令都有一个TransactionID，作为命令返回，也需要用同一个TransactionID以分辨对哪个命令的回应

| 字段                                   | 类型   | 类型                     |
| -------------------------------------- | ------ | ------------------------ |
| CommandName(命令名字)                  | String | 命令的名字               |
| TransactionID(事务ID)                  | Number |                          |
| CommandObject(命令包含的参数对象)      | Object | 键值对集合表示的命令参数 |
| OptionalUserArguments（额外的用户参数) | Object | 用户自定义的额外信息     |

如果消息中的TransactionID不为0的话，对端需要对该命令做出响应，响应的消息结构如下：

| 字段                | 类型   | 类型                                  |
| ------------------- | ------ | ------------------------------------- |
| CommandName(命令名) | String | 命令的名称                            |
| TransactionID       | Number | 上面接收到的命令消息中的TransactionID |
| CommandObject       | Object | 命令参数                              |
| OptionalArguents    | Object | 用户自定义参数                        |

#### 3.2.2.6.1.NetConnection Commands(连接层的命令)

代表服务端和客户端之间连接的更高层的对象。包含4个命令类型。

- connect：该命令是Client先发送给Server，意思是我要连接，能建立连接吗？ Server返回含`_result`或者`_error`命令名, 返回`_result`,表示server能提供服务，client可以进行下一步。`_error`，很明显Server端不能提供服务。
   消息结构如下：

connect命令抓包如下

![](/home/feng/Desktop/Screenshot from 2020-11-06 15-31-23.png)

在chunk body中有多种数据格式，比如以第一个参数String("connect")为例，首先读取一个字节，0x02表示这是一个string类型的数据，然后再读取两个字节表示这个string的长度，于是是7，然后再读取7个字节的数据转为string，就完成了对string(connect)的解析，因为后面还有字节，于是继续进行解析。直到完成。

大多数的body格式为一个string，即命令的名字，一个number，即TransactionID，一个object 即CommandObject（命令的参数）

amf0有多种数据类型

```c++
#define RTMP_AMF0_Number                     0x00
#define RTMP_AMF0_Boolean                     0x01
#define RTMP_AMF0_String                     0x02
#define RTMP_AMF0_Object                     0x03
#define RTMP_AMF0_MovieClip                 0x04 // reserved, not supported
#define RTMP_AMF0_Null                         0x05
#define RTMP_AMF0_Undefined                 0x06
#define RTMP_AMF0_Reference                 0x07
#define RTMP_AMF0_EcmaArray                 0x08
#define RTMP_AMF0_ObjectEnd                 0x09
#define RTMP_AMF0_StrictArray                 0x0A
#define RTMP_AMF0_Date                         0x0B
#define RTMP_AMF0_LongString                 0x0C
#define RTMP_AMF0_UnSupported                 0x0D
#define RTMP_AMF0_RecordSet                 0x0E // reserved, not supported
#define RTMP_AMF0_XmlDocument                 0x0F
#define RTMP_AMF0_TypedObject                 0x10
// AVM+ object is the AMF3 object.
#define RTMP_AMF0_AVMplusObject             0x11
// origin array whos data takes the same form as LengthValueBytes
#define RTMP_AMF0_OriginStrictArray         0x20

// User defined
#define RTMP_AMF0_Invalid                     0x3F
```



call：NetConnection 对象的调用方法在接收端运行远程过程调用。远程方法的名作为调用命令的参数。
   消息的结构如下：

| 字段                  | 类型   | 类型                                     |
| --------------------- | ------ | ---------------------------------------- |
| ProcedureName(进程名) | String | 要调用的进程名称                         |
| TransactionID         | Number | 如果想要对端响应的话置为非0值，否则置为0 |
| CommandObject         | Object | 命令参数                                 |
| OptionalArguents      | Object | 用户自定义参数                           |



- close：不知道为何协议里没写这个命令的内容，我猜应该是close connect。

createStream：客户端将此命令发送到服务器以创建用于消息通信的逻辑通道。音频，视频和元数据的发布是通过使用createStream命令创建的流通道进行的。NetConnection 本身是默认的流通道,具有流ID 0。协议和一少部分命令消息,包括创建流,就使用默认的通讯通道。


客户端给服务器发送

| 字段                | 类型   | 类型                                  |
| ------------------- | ------ | ------------------------------------- |
| CommandName(命令名) | String | “createStream”                        |
| TransactionID       | Number | 上面接收到的命令消息中的TransactionID |
| CommandObject       | Object | 命令参数                              |

服务器给客户端返回

| 字段                | 类型   | 类型                                  |
| ------------------- | ------ | ------------------------------------- |
| CommandName(命令名) | String | “createStream”                        |
| TransactionID       | Number | 上面接收到的命令消息中的TransactionID |
| CommandObject       | Object | 命令参数                              |
| Stream ID           | Number | 返回Stream ID，或者错误信息           |

#### 3.2.2.6.2.NetStream Commands(流连接上的命令)

Netstream建立在NetConnection之上，通过NetConnection的createStream命令创建，用于传输具体的音频、视频等信息。

 在传输层协议之上只能连接一个NetConnection，但一个NetConnection可以建立多个NetStream来建立不同的流通道传输数据。
 以下会列出一些常用的NetStream Commands，服务端收到命令后会通过onStatus的命令来响应客户端，表示当前NetStream的状态。
 onStatus命令的消息结构如下：

| 字段                | 类型   | 类型                                                         |
| ------------------- | ------ | ------------------------------------------------------------ |
| CommandName(命令名) | String | “onStatus”                                                   |
| TransactionID       | Number | 恒为0                                                        |
| CommandObject       | NULL   | 对onSatus命令来说不需要这个字段                              |
| InfoObject          | Object | AMF类型的Object，至少包含以下三个属性：1、“level”，String类型，可以为“warning”、”status”、”error”中的一种；2、”code”,String类型，代表具体状态的关键字,比如”NetStream.Play.Start”表示开始播流；3、”description”，String类型，代表对当前状态的描述，提供对当前状态可读性更好的解释，除了这三种必要信息，用户还可以自己增加自定义的键值对 |

- play(播放)

```ruby
         +-------------+                                     +----------+    
         | Play Client |                  |                  |  Server  |
         +-------------+                  |                  +----------+
                |          |Handshaking and Application|          |
                |          |         connect done      |          |
                |                         |                       |
       ---+---- |---------Command Message(createStream) --------->|
       Create   |                                                 |
       Stream   |                                                 |
       ---+---- |<-------------- Command Message -----------------|
                |       (_result- createStream response)          |
                |                                                 |
       ---+---- |------------ Command Message (play) ------------>|
         play   |                                                 |
          |     |<---------------- SetChunkSize ------------------|
          |     |                                                 |
          |     |<----- User Control (StreamIsRecorded) ----------|
          |     |                                                 |
          |     |<-------- UserControl (StreamBegin) -------------|
          |     |                                                 |
          |     |<---- Command Message(onStatus-play reset) ------|
          |     |                                                 |
          |     |<---- Command Message(onStatus-play start) ------|
          |     |                                                 |
          |     |------------------ Audio Message---------------->|
          |     |                                                 |
          |     |------------------ Video Message---------------->|
          |     |                                                 |
                                          |
                                          |
                 Keep receiving audio and video stream till finishes
```

```css
a. 客户端从服务端接收到流创建成功消息,发送播放命令到服务端。  
b. 接收到播放命令后,服务端发送协议消息设置块大小。  
c. 服务端发送另一个协议消息(用户控制消息),并且在消息中指定事件” streamisrecorded” 和流 ID 。消息承载的头 2 个字,为事件类型,后4 个字节为流 ID 。  
d. 服务端发送事件” streambegin” 的协议消息(用户控制),告知客户端流 ID 。 
e. 服务端发送响应状态命令消息`NetStream.Play.Start`&`NetStream.Play.reset` , 如果客户端发送的播放命令成功的话。只有当客户端发送的播放命令设置了 `reset`命令的条件下,服务端才发送`NetStream.Play.reset`消息。如果要发送的流 没有找的话,服务端发送`NetStream.Play.StreamNotFound`消息。在此之后服务端发送客户端要播放的音频和视频数据。 
```

- play命令的结构如下：

| 字段                | 类型    | 类型                                                         |
| ------------------- | ------- | ------------------------------------------------------------ |
| CommandName(命令名) | String  | “play”                                                       |
| TransactionID       | Number  | 恒为0                                                        |
| CommandObject       | NULL    | 不需要此字段，设为空                                         |
| StreamName          | String  | 要播放的流的名称                                             |
| 开始位置            | Number  | 可选参数，表示从何时开始播流，以秒为单位。默认为－2，代表选取对应该流名称的直播流，即当前正在推送的流开始播放，如果对应该名称的直播流不存在，就选取该名称的流的录播版本，如果这也没有，当前播流端要等待直到对端开始该名称的流的直播。如果传值－1，那么只会选取直播流进行播放，即使有录播流也不会播放；如果传值或者正数，就代表从该流的该时间点开始播放，如果流不存在的话就会自动播放播放列表中的下一个流 |
| 周期                | Number  | 可选参数，表示回退的最小间隔单位，以秒为单位计数。默认值为－1，代表直到直播流不再可用或者录播流停止后才能回退播放；如果传值为0，代表从当前帧开始播放 |
| 重置                | Boolean | 可选参数，true代表清除之前的流，重新开始一路播放，false代表保留原来的流，向本地的播放列表中再添加一条播放流 |

- play2(播放)

```undefined
和播放命令不同,play2命令可以切换到不同的码率,而不用改变已经播放的内容的时间线。
服务端对播放 2 命令可以请求的多个码率维护多个文件。
```

| 字段                | 类型   | 类型                                                         |
| ------------------- | ------ | ------------------------------------------------------------ |
| CommandName(命令名) | String | “play2”                                                      |
| TransactionID       | Number | 恒为0                                                        |
| CommandObject       | NULL   | 对onSatus命令来说不需要这个字段                              |
| parameters          | Object | AMF编码的Flash对象，包括了一些用于描述flash.net.NetstreamPlayOptions ActionScript obejct的参数 |

- deleteStream(删除流)

```undefined
当 NetStream 对象销毁的时候发送删除流命令。
```

| 字段                | 类型   | 类型                                   |
| ------------------- | ------ | -------------------------------------- |
| CommandName(命令名) | String | “deleteStream”                         |
| TransactionID       | Number | 恒为0                                  |
| CommandObject       | NULL   | 对onSatus命令来说不需要这个字段        |
| StreamID(流ID)      | Number | 本地已删除，不再需要服务器传输的流的ID |

- closeStream
- receiveAudio(接收音频)

```undefined
NetStream 对象发送接收音频消息通知服务端发送还是不发送音频到客户端。
```

| 字段                | 类型    | 类型                                                         |
| ------------------- | ------- | ------------------------------------------------------------ |
| CommandName(命令名) | String  | “receiveAudio”                                               |
| TransactionID       | Number  | 恒为0                                                        |
| CommandObject       | NULL    | 对onSatus命令来说不需要这个字段                              |
| BoolFlag            | Boolean | true表示发送音频，如果该值为false，服务器端不做响应，如果为true的话，服务器端就会准备接受音频数据，会向客户端回复NetStream.Seek.Notify和NetStream.Play.Start的Onstatus命令告知客户端当前流的状态 |

- receiveVideo(接收视频)

```undefined
NetStream 对象发送 receiveVideo 消息通知服务端是否发送视频到客户端。  
```

| 字段                | 类型    | 类型                                                         |
| ------------------- | ------- | ------------------------------------------------------------ |
| CommandName(命令名) | String  | “receiveVideo”                                               |
| TransactionID       | Number  | 恒为0                                                        |
| CommandObject       | NULL    | 对onSatus命令来说不需要这个字段                              |
| BoolFlag            | Boolean | true表示发送视频，如果该值为false，服务器端不做响应，如果为true的话，服务器端就会准备接受视频数据，会向客户端回复NetStream.Seek.Notify和NetStream.Play.Start的Onstatus命令告知客户端当前流的状态 |

- publish(推送数据)
   由客户端向服务器发起请求推流到服务器。

  ```ruby
           +-------------+                                     +----------+    
           |  Client     |                  |                  |  Server  |
           +-------------+                  |                  +----------+
                  |          |      Handshaking  Done    |          |
                  |                         |                       |
                  |                         |                       |
         ---+---- |---------   Command Message(connect)   --------->|
            |     |                                                 |
         Connect  |<---------- Window Acknowledge Size -------------|
            |     |                                                 |
            |     |<------------- Set Peer BandWidth ---------------|
            |     |                                                 |
            |     |----------- Window Acknowledge Size ------------>|
            |     |                                                 |
            |     |<--------- User Control(StreamBegin) ------------|
            |     |                                                 |
         ---+---- |--------------- Command Message ---------------->|
                  |          (_result- connect response)            |
                  |                                                 |
         ---+---- |---------Command Message(createStream) --------->|
         Create   |                                                 |
         Stream   |                                                 |
         ---+---- |<-------------- Command Message -----------------|
                  |       (_result- createStream response)          |
                  |                                                 |
         ---+---- |--------- Command Message (publish) ------------>|
            |     |                                                 |
         publish  |<-------- UserControl (StreamBegin) -------------|
            |     |                                                 |
            |     |---------- Data Message (Metadata) ------------->|
            |     |                                                 |
            |     |------------------ Audio Message---------------->|
            |     |                                                 |
            |     |----------------- SetChunkSize ----------------->|
            |     |                                                 |
            |     |<--------------- Command Message ----------------|
            |     |            (_result- publish result)            |
            |     |------------------ Video Message---------------->|
                                           |
                                           |
                             Until the stream is complete
  ```

```undefined
     客户端发送一个发布命令,发布一个命名流到服务端。使用这个名字,任何客户端可以播放该流并且接收音频,视频,和数据消息。  
```

- publish命令结构如下：

| 字段                         | 类型   | 类型                                                         |
| ---------------------------- | ------ | ------------------------------------------------------------ |
| CommandName(命令名)          | String | “publish”                                                    |
| TransactionID                | Number | 恒为0                                                        |
| CommandObject                | NULL   | 对onSatus命令来说不需要这个字段                              |
| PublishingName（推流的名称） | String | 流名称                                                       |
| PublishingType（推流类型）   | String | “live”、”record”、”append”中的一种。live表示该推流文件不会在服务器端存储；record表示该推流的文件会在服务器应用程序下的子目录下保存以便后续播放，如果文件已经存在的话删除原来所有的内容重新写入；append也会将推流数据保存在服务器端，如果文件不存在的话就会建立一个新文件写入，如果对应该流的文件已经存在的话保存原来的数据，在文件末尾接着写入 |

- seek(定位流的位置)
   定位到视频或音频的某个位置，以毫秒为单位。

```undefined
     客户端发送搜寻命令在一个媒体文件中或播放列表中搜寻偏移。 
```

- seek命令的结构如下：

| 字段                | 类型   | 类型                            |
| ------------------- | ------ | ------------------------------- |
| CommandName(命令名) | String | “seek”                          |
| TransactionID       | Number | 恒为0                           |
| CommandObject       | NULL   | 对onSatus命令来说不需要这个字段 |
| milliSeconds        | Number | 定位到该文件的xx毫秒处          |

- pause(暂停)
   客户端告知服务端停止或恢复播放

```undefined
     客户端发送暂停命令告诉服务端暂停或开始一个命令。 
```

- pause命令的结构如下：

| 字段                | 类型    | 类型                             |
| ------------------- | ------- | -------------------------------- |
| CommandName(命令名) | String  | “pause”                          |
| TransactionID       | Number  | 恒为0                            |
| CommandObject       | NULL    | 对onSatus命令来说不需要这个字段  |
| Pause/Unpause Flag  | Boolean | true表示暂停，false表示恢复      |
| milliSeconds        | Number  | 暂停或者恢复的时间，以毫秒为单位 |

如果Pause为true即表示客户端请求暂停的话，服务端暂停对应的流会返回NetStream.Pause.Notify的onStatus命令来告知客户端当前流处于暂停的状态，当Pause为false时，服务端会返回NetStream.Unpause.Notify的命令来告知客户端当前流恢复。如果服务端对该命令响应失败，返回_error信息。

#### 3.2.2.7.聚合消息

- <font size=3 color=#666666 >**MT=22**, Aggregate message, 聚合消息是含有一个消息列表的一种消息。消息类型值 22 ,保留用于聚合消息。
   </font>

```ruby
  +---------+-------------------------+  
  | Header  | Aggregate Message body  |  
  +---------+-------------------------+  
          聚合消息的格式
  +--------+--------------+--------------+--------+-------------+---------------+ - - - -
  |Header 0|Message Data 0|Back Pointer 0|Header 1|Message Data 1|Back Pointer 1|
  +--------+--------------+--------------+--------+--------------+--------------+ - - - -
            聚合消息的body  
```

```undefined
Back Pointer包含了前面消息的大小（包括Header的大小）。这个设置匹配了 flv 文件格式,可用于后向搜索。 
```

#### 3.2.3.接收命令消息反馈结果 ResponseCommand

通过块消息携带的数据，拼接成消息内容，通过AMF解读消息内容，略过不细讲

```cpp
#define RTMPConnectSuccess      @"NetConnection.Connect.Success"
#define RTMPPublishStart        @"NetStream.Publish.Start"
#define RTMPPublishBadName      @"NetStream.Publish.BadName"
#define RTMPPlayStart           @"NetStream.Play.Start"
#define RTMPPlayReset           @"NetStream.Play.Reset"
#define RTMPPlayStreamNotFound  @"NetStream.Play.StreamNotFound"

typedef enum : char {
    RTMPResponseCommand_Result       = 0x1, //_Result命令
    RTMPResponseCommandOnBWDone      = 0x2, //OnBWDone命令
    RTMPResponseCommandOnFCPublish   = 0x3, //OnFCPublish命令
    RTMPResponseCommandOnStatus      = 0x4, //OnStatus命令
    RTMPResponseCommandOnFCUnpublish = 0x5, //OnFCUnpublish命令
    RTMPResponseCommandOnMetaData    = 0x6, //OnMetaData命令
    RTMPResponseCommandUnkonwn       = 0x7f,//未知类型
    
} RTMPResponseCommandType;
```

<font size=4 color=000000>**附:**FLV数据封装简单描述</font>

```objectivec
FLV 由 FLV Header + FLV Body 组成

             -- FLV Body ------------------------------ ... ----------------------------
           /                                                                             \
+---------+-------------+-------------+-------------+-- ... --+-------------+-------------+
|   FLV   |    PreTag   |    Tag      |    PreTag   |         |    Tag      |    PreTag   | 
| Hearder |     Size    |     1       |     Size    |         |     N       |     Size    |
+---------+-------------+-------------+-------------+-- ... --+-------------+-------------+jiantou
                              |
┏—————————————————————————————┛
|
|  FLV Header占9bytes,flv的类型、版本的信息，
|  组成：Signature(3bytes)+Version(1bytes)+Flags(1bytes)+DataOffset(4bytes)
|      ①Signature:固定FLV三个字符作为标示，一般前3个字符为FLV是就认为是flv文件
|      ②Version:表示FLV的版本号
|      ③Flags:内容标示，第0位和第2位，分别表示video与audio存在的情况(1表示存在，0表示不存在) 
|           如0x05(00000101)同时存在视频、音频
|      ④DataOffset:表示FLV的Header长度。这里固定是9
|
|  FLV Body: TagSize0 + Tag1 + Tag1 Size + Tag2 + Tag2 Size + ... + TagN + TagN Size
|
|  PreTagSize占4bytes,值表示前一个Tag的长度
|  Tag 由 Tag Header + Tag Data 组成，Tag分三种类型，video、audio、scripts(元数据)
|
|     (某一个Tag)
┗——→ +------------+
     | Tag Header |
     +------------+
     | Tag Data   |
     +------------+

           +------------+------------+------------+------------+------------+
 TagHeader | Tag type   |  Data Size | Time Stamp |ExtTimeStamp|  Stream ID |
           +------------+------------+------------+------------+------------+
    
              +------------+------------+------------+-----------------------------------+
TagData_Video |    Frame   |    AvcPk   |Composition | AVC Pk Type == 0: SPS_PPS         |
              |    Codec   |    Type    |   Time     | AVC Pk Type == 1: nalu len + nalu |
              +------------+------------+------------+-----------------------------------+    
     
     
              +------------+------------+-----------------------------------+
TagData_Audio |   Format   |    AACPk   | AAC Pk Type == 0: AAC Spec        |
              |  SR,SZ,ST  |    Type    | AAC Pk Type == 1: AAC RAW         |
              +------------+------------+-----------------------------------+     
     

//AVC Pk Type
typedef NS_ENUM(NSUInteger, flv_video_h264_packettype)
{
    flv_video_h264_packettype_seqHeader     = 0,
    flv_video_h264_packettype_nalu          = 1,
    flv_video_h264_packettype_endOfSeq      = 2,
};

//AAC Pk Type
typedef NS_ENUM(NSUInteger, flv_audio_aac_packettype)
{
    msp_flv_audio_aac_packettype_seqHeader      = 0,
    msp_flv_audio_aac_packettype_raw            = 1,
};

  
  flv要求发送音视频之前先发送一个tag_scripts类型元数据，描述视频或音频的信息的数据，如宽度、高度、采样、声道、频率、编码等等
  flv要求第一个tag_video必须是sps pps数据包 之后才能发送视频编码数据
  flv要求第一个tag_audio必须是音频配置数据包 之后才能发送音频编码数据
  
  Tag Header占11bytes，存放当前Tag类型、TagData(数据区)的长度等信息
  +------------+
  | Tag Header |
  +------------+
      ①Tag类型:1byte，8 = 音频，9 = 视频，18(0x12) = 脚本， 其他 = 保留
      ②数据区长度(tag data size):3bytes
      ③时间戳:3bytes，整数单位毫秒，脚本类型Tag为0
      ④时间戳扩展:1bytes
      ⑤StreamsID:3bytes 总是0

      
  Tag Data 数据区，音频数据(TagData_Audio)、视频数据(TagData_Video)、脚本数据(TagData_Scripts)
  +------------+
  |  Tag Data  |
  +------------+
  
  TagData_Video
     第一个byte视频信息 = 帧类型(4bit) + 编码ID(4bit)
     后面数据，视频格式为 AVC(H.264)的话，后面为4个字节信息，AVCPacketType(1byte)和 CompositionTime(3byte)
           AVCPacketType == 1 则CompositionTime = Composition time offset
                后面三个字节也是0，说明这个tag记录的是AVCDecoderConfigurationRecord。包含sps和pps数据。
           后面数据为：0x01+sps[1]+sps[2]+sps[3]+0xFF+0xE1+sps_size+sps+01+pps_size+pps
           
           AVCPacketType != 1 则CompositionTime = 0
                后面三个字节也是0，说明这个tag记录的是AVCDecoderConfigurationRecord。包含sps和pps数据
           后面数据为：0x01+sps[1]+sps[2]+sps[3]+0xFF+0xE1+sps_size+sps+01+pps_size+pps
  
  
  TagData_Audio
     第一个byte音频信息 = 音频格式(4bit) + 采样率(2bit) + 采样长度(1bit) + 音频类型(1bit)
     后面数据，如果音频格式为AAC，后面跟1byte 0x00/0x01。
           如果0x00 后面跟 audio config data 数据 需要作为第一个 audio tag 发送
           如果0x01 后面跟 audio frame data 数据
  
  
  TagData_Scripts
     数据类型 +（数据长度）+ 数据，数据类型占1byte,数据长度根据数据类型
     数据类型: 0 = Number type
              1 = Boolean type
              2 = String type
              3 = Object type
              4 = MovieClip type
              5 = Null type
              6 = Undefined type
              7 = Reference type
              8 = ECMA array type
              10 = Strict array type
              11 = Date type
              12 = Long string type
      如果为 String type ,那么数据长度占2bytes(Long string type 占 4bytes)，后面就是字符串数据
            举个栗子：0x02(String 类型)+0x000a("onMetaData"长度) + "onMetaData"
      如果为 Number type ,没有数据长度，后面直接为8bytes的 Double 类型数据
      如果为 Boolean type,没有数据长度，后面直接为1byte的 Bool 类型数据
      如果为 ECMA array type,数据长度占4bytes 值表示数组长度，后面 键是 String 类型的，开头0x02被省略，
            直接跟字符串长度，然后是字符串，在是值类型（根据上面来）
  
#pragma mark - video Tag Data

// 帧类型 占4bits 这里只用到了 key、inner
typedef NS_ENUM (NSUInteger, flv_video_frametype)
{
    flv_video_frametype_key         = 1,// keyframe (for AVC, a seekable frame)
    flv_video_frametype_inner       = 2,// inter frame (for AVC, a non-seekable frame)
    //flv_video_frametype_inner_d     = 3,// disposable inter frame (H.263 only)
    //flv_video_frametype_key_g       = 4,// generated keyframe (reserved for server use only)
    //flv_video_frametype_vi_cf       = 5,// video info/command frame
};

// 编码(格式)ID 占4bits 只用到了H264
typedef NS_ENUM (NSUInteger, flv_video_codecid)
{
    flv_video_codecid_JPEG          = 1,// JPEG (currently unused)
    flv_video_codecid_H263          = 2,// Sorenson H.263
    //flv_video_codecid_ScreenVideo   = 3,// Screen video
    //flv_video_codecid_On2VP6        = 4,// On2 VP6
    //flv_video_codecid_On2VP6_AC     = 5,// On2 VP6 with alpha channel
    //flv_video_codecid_ScreenVideo2  = 6,// Screen video version 2
    flv_video_codecid_H264          = 7,// AVC
    //flv_video_codecid_RealH263      = 8,
    //flv_video_codecid_MPEG4         = 9,
};


#pragma mark - audio Tag Data

// 音频编码(音频格式)ID 占4bits 只用到了AAC
typedef NS_ENUM (NSUInteger, flv_audio_codecid)
{
    //flv_audio_codecid_PCM           = 0,// Linear PCM, platform endian
    //flv_audio_codecid_ADPCM         = 1,// ADPCM
    flv_audio_codecid_MP3           = 2,// MP3
    //flv_audio_codecid_PCM_LE        = 3,// Linear PCM, little endian
    //flv_audio_codecid_N16           = 4,// Nellymoser 16-kHz mono
    //flv_audio_codecid_N8            = 5,// Nellymoser 8-kHz mono
    //flv_audio_codecid_N             = 6,// Nellymoser
    //flv_audio_codecid_PCM_ALAW      = 7,// G.711 A-law logarithmic PCM
    //flv_audio_codecid_PCM_MULAW     = 8,// G.711 mu-law logarithmic PCM
    //flv_audio_codecid_RESERVED      = 9,// reserved
    flv_audio_codecid_AAC           = 10,// AAC
    //flv_audio_codecid_SPEEX         = 11,// Speex
    //flv_audio_codecid_MP3_8         = 14,// MP3 8-Khz
    //flv_audio_codecid_DSS           = 15,// Device-specific sound
};

// soundSize 8bit/16bit 采样长度 压缩过的音频都是16bit  占1bit
typedef NS_ENUM (NSUInteger, flv_audio_soundsize)
{
    flv_audio_soundsize_8bit        = 0,// snd8Bit
    flv_audio_soundsize_16bit       = 1,// snd16Bit
};

// sound rate 5.5 11 22 44 kHz 采样率 对于AAC总是3
typedef NS_ENUM (NSUInteger, flv_audio_soundrate)
{
    flv_audio_soundrate_5_5kHZ      = 0,// 5.5-kHz
    flv_audio_soundrate_11kHZ       = 1,// 11-kHz
    flv_audio_soundrate_22kHZ       = 2,// 22-kHz
    flv_audio_soundrate_44kHZ       = 3,// 44-kHz
};

// sound type mono/stereo  对于AAC总是1 立体音
typedef NS_ENUM (NSUInteger, flv_audio_soundtype)
{
    flv_audio_soundtype_mono        = 0,// sndMono
    flv_audio_soundtype_stereo      = 1,// sndStereo
};  
```