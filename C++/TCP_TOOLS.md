使用命令

`nl -l 9090`  服务端监听9090端口

`nc 127.0.0.1 9090` 客户端连接9090端口

`sudo tcpdump -i any port 9090 -vv`  监视9090端口的数据传输。



```shell
tcpdump: listening on any, link-type LINUX_SLL (Linux cooked v1), capture size 262144 bytes


19:19:23.828355 IP (tos 0x0, ttl 64, id 31218, offset 0, flags [DF], proto TCP (6), length 60)
    localhost.41094 > localhost.9090: Flags [S], cksum 0xfe30 (incorrect -> 0x84cc), seq 712574881, win 65495, options [mss 65495,sackOK,TS val 917610081 ecr 0,nop,wscale 7], length 0
    
19:19:23.828367 IP (tos 0x0, ttl 64, id 0, offset 0, flags [DF], proto TCP (6), length 60)
    localhost.9090 > localhost.41094: Flags [S.], cksum 0xfe30 (incorrect -> 0xae5e), seq 3456512847, ack 712574882, win 65483, options [mss 65495,sackOK,TS val 917610081 ecr 917610081,nop,wscale 7], length 0
    
19:19:23.828376 IP (tos 0x0, ttl 64, id 31219, offset 0, flags [DF], proto TCP (6), length 52)
    localhost.41094 > localhost.9090: Flags [.], cksum 0xfe28 (incorrect -> 0xd51a), seq 1, ack 1, win 512, options [nop,nop,TS val 917610081 ecr 917610081], length 0
```

### Internet Header Format

```
    0                   1                   2                   3           
    0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |Version|  IHL  |Type of Service|          Total Length         |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |         Identification        |Flags|      Fragment Offset    |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |  Time to Live |    Protocol   |         Header Checksum       |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |                       Source Address                          |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |                    Destination Address                        |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |                    Options                    |    Padding    |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```
### TCP Header Format

```
    0                   1                   2                   3   
    0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |          Source Port          |       Destination Port        |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |                        Sequence Number                        |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |                    Acknowledgment Number                      |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |  Head |           |U|A|P|R|S|F|                               |
   | Length| Reserved  |R|C|S|S|Y|I|            Window             |
   |       |           |G|K|H|T|N|N|                               |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |           Checksum            |         Urgent Pointer        |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |                    Options                    |    Padding    |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |                             data                              |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

​    **16位端口号**：告知主机该报文段是来自哪里（源端口Source Port）以及传给哪个上层协议或应用程序（目的端口Destination  Port）的。进行TCP通信时，客户端通常使用系统自动选择的临时端口号，而服务器则使用知名服务端口号（比如DNS协议对应端口53，HTTP协议对应80，这些端口号可在/etc/services文件中找到）。

​    **32位序号**：一次TCP通信（从TCP连接建立到断开）过程中某一个传输方向上的字节流的每个字节的编号。假设主机A和主机B进行TCP通信，A发送给B的第一个TCP报文段中，序号值被系统初始化为某个随机值ISN（Initial Sequence  Number，初始序号值）。那么在该传输方向上（从A到B），后续的TCP报文段中序号值将被系统设置成ISN加上该报文段所携带数据的第一个字节在整个字节流中的偏移。例如，某个TCP报文段传送的数据是字节流中的第1025~2048字节，那么该报文段的序号值就是ISN+1025.另外一个传输方向（从B到A）的TCP报文段的序号值也具有相同的含义。

​    **32位确认号（acknowledgement number）**：用作对另一方发送来的TCP报文段的响应。其值是收到的TCP报文段的序号值加1。假设主机A和主机B进行TCP通信，那么A发送出的TCP报文段不仅携带自己的序号，而且包含对B发送来的TCP报文段的确认号。反之，B发送出的TCP报文段也同时携带自己的序号和对A发送来的报文段的确认号。（实际上就是ACK值），当初次建立连接时，如A要连接B，a第一次连接不知道b的序列号，所以会发全零。

​    **4位头部长度（head length）**：标识该TCP头部有多少个32bit字（4字节）。因为4位最大能标识15，所以TCP头部最长是60字节。

​    **6位标志位包含如下几项**：

​    URG标志，表示紧急指针（urgent pointer）是否有效。

​    ACK标志，表示确认号是否有效。我们称携带ACK标识的TCP报文段为确认报文段。

​    PSH标志，提示接收端应用程序应该立即从TCP接收缓冲区中读走数据，为接收后续数据腾出空间（如果应用程序不将接收

​    到的数据读走，它们就会一直停留在TCP接收缓冲区中）。

​     RST标志，表示要求对方重新建立连接。我们称携带RST标志的TCP报文段为复位报文段。

​     SYN标志，表示请求建立一个连接。我们称携带SYN标志的TCP报文段为同步报文段。

​     FIN标志，表示通知对方本端要关闭连接了。我们称携带FIN标志的TCP报文段为结束报文段。

​	此外还有六位保留存在

​    **16位窗口大小（window size）**：是TCP流量控制的一个手段。这里说的窗口，指的是接收通告窗口（Receiver Window，RWND）。它告诉对方本端的TCP接收缓冲区还能容纳多少字节的数据，这样对方就可以控制发送数据的速度。

​    **16位校验和（TCP check sum）：**由发送端填充，接收端对TCP报文段执行CRC算法以检验TCP报文段在传输过程中是否损坏。注意，这个校验不仅包括TCP头部，也包括数据部分。这也是TCP可靠传输的一个重要保障。

​    **16位紧急指针（urgent pointer）**：是一个正的偏移量。它和序号字段的值相加表示最后一个紧急数据的下一字节的序号。因此，确切地说，这个字段是紧急指针相对当前序号的偏移，不妨称之为紧急偏移。TCP的紧急指针是发送端向接收端发送紧急数据的方法。



Options最多为40字节数据，因为TCP头长度最多为60字节，有20字节有固定格式的



# TCP输出格式

 `时.分:秒 IP  src > dst: Flags [tcpflags], seq data-seqno, ack ackno, win window, urg urgent, options [opts], length len`

 IP 

tos表示服务类型，4bit的tos分别表示最小时延，最大吞吐量，最高可靠性，最小费用。这里都是0，表示一般服务，其余4bit废用，置0.

TTL(time - to - live)生存时间字段设置了数据报可以经过的最多路由器数。它指定了数据报的生存时间。TTL的初始值由源主机设置（通常为32或64），一旦经过一个处理它的路由器，它的值就减去1。当该字段的值为0时，数据报就被丢弃，并发送ICMP报文通知源主机。

id 对应IP报文头的Identification,用于IP分片重组。

offset 也用于IP分片重组，表示相对于原始未分片的报文的位置。

flags MF表示有更多分片，DF表示不分片，这里是DF，未使用分片，所以id和offset的值都可以忽略。

proto 表示协议，可以是TCP，UDP等，这里是TCP。

length 总长度字段，是指整个I P数据报的长度(至于首部长度这里没有给出，首部长度给出首部中32 bit字的数目。需要这个值是因为任选字段的长度是可变的。这个字段占4 bit，因此TCP最多有60字节的首部。然而没有任选字段，正常的长度是20字节)。
 标志位
S (SYN), F (FIN), P (PUSH), R (RST), U (URG), W (ECN CWR), E (ECN-Echo) or "." (ACK), or 'none' if no flags are set.

tcp连接

```

19:19:23.828355 IP (tos 0x0, ttl 64, id 31218, offset 0, flags [DF], proto TCP (6), length 60)
    localhost.41094 > localhost.9090: Flags [S], cksum 0xfe30 (incorrect -> 0x84cc), seq 712574881, win 65495, options [mss 65495,sackOK,TS val 917610081 ecr 0,nop,wscale 7], length 0

19:19:23.828367 IP (tos 0x0, ttl 64, id 0, offset 0, flags [DF], proto TCP (6), length 60)
    localhost.9090 > localhost.41094: Flags [S.], cksum 0xfe30 (incorrect -> 0xae5e), seq 3456512847, ack 712574882, win 65483, options [mss 65495,sackOK,TS val 917610081 ecr 917610081,nop,wscale 7], length 0
    
19:19:23.828376 IP (tos 0x0, ttl 64, id 31219, offset 0, flags [DF], proto TCP (6), length 52)
    localhost.41094 > localhost.9090: Flags [.], cksum 0xfe28 (incorrect -> 0xd51a), seq 1, ack 1, win 512, options [nop,nop,TS val 917610081 ecr 917610081], length 0

```

![](/home/feng/Desktop/20180830152422831.png)

一般都是客户端向服务器发送链接，

* 首先客户端发送一个初始序号ISN，即图片中的seq ,并设置SYN标志位为1

* 然后服务器对客户端请求做应答，设置ACK为seq+1，并发送请求连接客户端，设置SYN标志位为1，并发送自己的序列号

* 当客户端收到信息后。又给服务端发送确认信息，即ACK请求，

在握手阶段，也设置了WIN窗口大小

win  TCP窗口大小，通知对方，发送方最多还可以接收的数据量，用于TCP的拥塞控制。第一个报文表示客户端通知服务端，客户端可以接受的数据的缓存区最大是65495个字节。服务端通知客户端，服务端最多可以接受的缓存区最大是65483，这个窗口大小在一方接受数据，却没有read的时候，窗口会逐渐减小，直至为0，最后对方不可以发送任何数据(如果要做该测试，需要发送的数据量大概接近65535，因为窗口的缓存区也会在剩余容量减小时，自动增加总共容量，直到总共容量接近65535，接下来就会看到win越来越小，直至0)。

但是在tcpdump中可以看到，事实上客户端第二次发送数据给服务端时将seq 1, ack 1, win 512。实际上这里只是方便查看，ip底层的数据还是如图上所示，seq=x+1， ack= y+1。 

`sudo tcpdump -i any port 9090 -vv -XX`



服务器发送数据`qw`

```shell
21:05:06.250696 IP (tos 0x0, ttl 64, id 33245, offset 0, flags [DF], proto TCP (6), length 55)
localhost.9090 > localhost.44078: Flags [P.], cksum 0xfe2b (incorrect -> 0x5ad8), seq 1:4, ack 1, win 512, options [nop,nop,TS val 923952504 ecr 923831085], length 3
21:05:06.250738 IP (tos 0x0, ttl 64, id 63700, offset 0, flags [DF], proto TCP (6), length 52)
    localhost.44078 > localhost.9090: Flags [.], cksum 0xfe28 (incorrect -> 0xfc0a), seq 1, ack 4, win 512, options [nop,nop,TS val 923952504 ecr 923952504], length 0
```



客户端发送数据`12`

```shell
21:06:08.410208 IP (tos 0x0, ttl 64, id 63701, offset 0, flags [DF], proto TCP (6), length 55)
 localhost.44078 > localhost.9090: Flags [P.], cksum 0xfe2b (incorrect -> 0xcdfd), seq 1:4, ack 4, win 512, options [nop,nop,TS val 924014663 ecr 923952504], length 3
21:06:08.410248 IP (tos 0x0, ttl 64, id 33246, offset 0, flags [DF], proto TCP (6), length 52)
    localhost.9090 > localhost.44078: Flags [.], cksum 0xfe28 (incorrect -> 0x1668), seq 4, ack 4, win 512, options [nop,nop,TS val 924014663 ecr 924014663], length 0
```

由于发送时候只发了两个字符加上\n，一个长度为3