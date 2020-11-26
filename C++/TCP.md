一、黏包成因
1、tcp协议的拆包机制
当发送端缓冲区的长度大于网卡的MTU时，tcp会将这次发送的数据拆成几个数据包发送出去。 
MTU是Maximum Transmission Unit的缩写。意思是网络上传送的最大数据包。MTU的单位是字节。 
大部分网络设备的MTU都是1500。如果本机的MTU比网关的MTU大，大的数据包就会被拆开来传送，
这样会产生很多数据包碎片，增加丢包率，降低网络速度。


2、tcp的合包机制
TCP（transport control protocol，传输控制协议）是面向连接的，面向流的，提供高可靠性服务。
收发两端（客户端和服务器端）都要有一一成对的socket，因此，发送端为了将多个发往接收端的包，更有效的发到对方，
使用了优化方法（Nagle算法），将多次间隔较小且数据量小的数据，合并成一个大的数据块，然后进行封包。
但是这样，接收端，就难于分辨出来了，必须提供科学的拆包机制。 即面向流的通信是无消息保护边界的。 
对于空消息：tcp是基于数据流的，于是收发的消息不能为空，这就需要在客户端和服务端都添加空消息的处理机制，防止程序卡住，
而udp是基于数据报的，即便是你输入的是空内容（直接回车），也可以被发送，udp协议会帮你封装上消息头发送过去。 
可靠黏包的tcp协议：tcp的协议数据不会丢，没有收完包，下次接收，会继续上次继续接收，己端总是在收到ack时才会清除缓冲区内容。数据是可靠的，但是会粘包。


3、说明
发送端可以是一K一K地发送数据，而接收端的应用程序可以两K两K地提走数据，当然也有可能一次提走3K或6K数据，或者一次只提走几个字节的数据。
也就是说，应用程序所看到的数据是一个整体，或说是一个流（stream），一条消息有多少字节对应用程序是不可见的，因此TCP协议是面向流的协议，这也是容易出现粘包问题的原因。
而UDP是面向消息的协议，每个UDP段都是一条消息，应用程序必须以消息为单位提取数据，不能一次提取任意字节的数据，这一点和TCP是很不同的。
怎样定义消息呢？可以认为对方一次性write/send的数据为一个消息，需要明白的是当对方send一条信息的时候，无论底层怎样分段分片，TCP协议层会把构成整条消息的数据段排序完成后才呈现在内核缓冲区。


也就是：
用UDP协议发送时，用sendto函数最大能发送数据的长度为：65535- IP头(20) – UDP头(8)＝65507字节。用sendto函数发送数据时，如果发送数据长度大于该值，
则函数会返回错误。（丢弃这个包，不进行发送） 

用TCP协议发送时，由于TCP是数据流协议，因此不存在包大小的限制（暂不考虑缓冲区的大小），这是指在用send函数时，数据长度参数不受限制。
而实际上，所指定的这段数据并不一定会一次性发送出去，如果这段数据比较长，会被分段发送，如果比较短，可能会等待和下一次数据一起发送。




例如：
基于tcp的套接字客户端往服务端上传文件，发送时文件内容是按照一段一段的字节流发送的，在接收方看了，根本不知道该文件的字节流从何处开始，在何处结束
此外，发送方引起的粘包是由TCP协议本身造成的，TCP为提高传输效率，发送方往往要收集到足够多的数据后才发送一个TCP段。若连续几次需要send的数据都很少，
通常TCP会根据优化算法把这些数据合成一个TCP段后一次发送出去，这样接收方就收到了粘包数据。



上代码：
服务端：
import socket
sk = socket.socket()
sk.setsockopt(socket.SOL_SOCKET,socket.SO_REUSEADDR,1)
sk.bind(('127.0.0.1',8000))
sk.listen()

conn,addr = sk.accept()
ret = conn.recv(1024)
print(ret.decode('utf-8'))
conn.close()
sk.close()


客户端：
import socket
sk = socket.socket()
sk.connect(('127.0.0.1',8000))
sk.send(b'hello,')
sk.send(b'world,')
sk.send(b'hi')
sk.close()


结果：
hello,world,hi

解释：
正常来说，一个send必须对应一个recv，
但是我们都知道python程序是由上至下执行的，那么：
sk.send(b'hello,')
sk.send(b'world,')
sk.send(b'hi')
上面这三句代码几乎在一瞬间就执行了，而由于要发送的数据很小，而且是时间间隔很短，
发送方就会把这几条数据合成一条数据，再发送过去，在接收端其实收到的就是一次传来的数据，
所以这个时候三次send，对应一次recv，这就是黏包。


4、总结
黏包现象只发生在tcp协议中：
1.从表面上看，黏包问题主要是因为发送方和接收方的缓存机制、tcp协议面向流通信的特点。
2.实际上，主要还是因为接收方不知道消息之间的界限，不知道一次性提取多少字节的数据所造成的


合包现象
    数据很短
    时间间隔短
拆包现象
    大数据会发生拆分
    不会一次性的全部发送到对方
    对方在接受的时候很可能没有办法一次性接收到所有的信息
    那么没有接受完的信息很可能和后面的信息黏在一起
粘包现象只发生在tcp协议
    tcp协议的传输 是 流式传输
    每一条信息与信息之间是没有边界的

udp协议中是不会发生粘包现象的
    适合短数据的发送
    不建议你发送过长的数据
    数据过长会增大你数据丢失的几率

在程序中会出现粘包：收发数据的边界不清晰
接收数据这一端不知道要接收数据的长度到底是多少





二、黏包解决方案
1、解决方案一
问题的根源在于，接收端不知道发送端将要传送的字节流的长度，所以解决粘包的方法就是围绕，如何让发送端在发送数据前，
把自己将要发送的字节流总大小让接收端知晓，然后接收端来一个死循环接收完所有数据。

就是说：
如果你要发送一个数据----hello，它是5个字节的，
你在接收端设置了只接收5个字节，那么就算发生黏包也没关系，
因为你只接收了5个字节，黏在一起的剩下的数据也就没有读取到了。
send(b'hello')   ----->   recv(5)

那么我们就有了一个思路，就是在发送消息的时候，我们主动告诉接收端我们要发送的数据的长度，
接收端按照接收的长度来接收数据。例如：
发送端：
send(b'5hello')  


接收端：
num = recv(1)  # 代表接收第一个字节，也就是只把长度5接收了
num_len = int(num.decode('utf-8'))  # 把长度的类型转成整型
msg = recv(num_len)  # 按照长度接收数据


代码：
服务端：
import socket
sk = socket.socket()
sk.setsockopt(socket.SOL_SOCKET,socket.SO_REUSEADDR,1)
sk.bind(('127.0.0.1',8001))
sk.listen()

conn,addr = sk.accept()
conn.send(b'5hello')
conn.send(b'2hi')

conn.close()
sk.close()


客户端：
import socket
sk = socket.socket()
sk.connect(('127.0.0.1',8001))

num = sk.recv(1)
num_len = int(num.decode('utf-8'))
msg1 = sk.recv(num_len)
print(msg1)

num2 = sk.recv(1)
num2_len = int(num2.decode('utf-8'))
msg2 = sk.recv(num2_len)
print(msg2)

sk.close()

结果：
hello
hi

但是这样写每次只能接收个位数的数据，我们可以把长度设置成4个长度，即0000-9999
发送端：
send(b'0005hello')  


接收端：
num = recv(4)  # 代表接收前四个字节，也就是只把长度0005接收了
num_len = int(num.decode('utf-8'))  # 把长度的类型转成整型
msg = recv(num_len)  # 按照长度接收数据


但实际中，我们要传的数据往往很大的而这种方式虽然能解决一些问题，但是这样写一次也最多发送9999个字节(大概9.7KB)，
那么如果2G的东西就要发送大概21万次循环才能发送完。


补充一个字符串的方法zfill：在左边给字符补0
print('1'.zfill(4))   # 0001



2、解决方案2
首先介绍一个模块struct：该模块可以把一个类型，如数字，转成固定长度(4)的bytes
import struct
ret1 = struct.pack('i',10238976)    # i代表把整型的数据转换成bytes类型的数据
ret2 = struct.pack('i',1)

print(ret1,len(ret1))  # b'\x00<\x9c\x00'  4
print(ret2,len(ret2))  # b'\x01\x00\x00\x00' 4
可以看到：数字10238976转成bytes后，长度为4，数字1转成bytes后，长度也是为4。

num1 = struct.unpack('i',ret1)   # unpack把bytes类型转成第一个参数代表的类型(这里是i，也就是int 整型，但返回的是一个元组)
print(num1)  # (10238976,)  元组
print(num1[0])  # 10238976 取元组的第一个值即可

注意:'i' 所能转换的数字范围是 -2147483648 <= number <= 2147483647 
超出这个范围就会报错，就是不能这样写  struct.pack('i',2147483648)



代码：
服务端：
import socket
import struct
sk = socket.socket()
sk.setsockopt(socket.SOL_SOCKET,socket.SO_REUSEADDR,1)
sk.bind(('127.0.0.1',8002))
sk.listen()

conn,addr = sk.accept()
while True:
    msg = input('>>>:').encode('utf-8')  # 要发送的内容
    pack_num = struct.pack('i',len(msg))  # 计算内容的长度
    conn.send(pack_num)  
    conn.send(msg)
conn.close()
sk.close()




客户端：
import socket
import struct

sk = socket.socket()
sk.connect(('127.0.0.1',8002))

while True:
    pack_num = sk.recv(4)
    num = struct.unpack('i',pack_num)[0]
    ret = sk.recv(num)
    print(ret.decode('utf-8'))
sk.close()
