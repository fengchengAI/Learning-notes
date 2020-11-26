## <center>ASIO<center/> 



deadline_timer 时间事件

```cpp
int main(){
    io_service io; 
    //io_service 是程序和电脑交互的接口,一般在ASIO的程序中,都要有一个io_service对象. 
    deadline_timer t(io,boost::posix_time::seconds(5));
    //deadline_timer是一个事件模型,该模型中第一个参数为io_service,第二个参数为boost::posix_time类(不能使用std的chrono)
    t.async_wait([](boost::system::error_code){cout<<1<<2<<endl;});
    //async_wait是一个异步事件,该事件会注册一个函数,在上面例子中为一个简单地lamnda函数,这里只是注册事件,运行是在io.run被调用时才开始的.
    //io可以看成是一个事件池,async_wait会将事件注册进去,当run后会运行池中的东西,运行完成后会返回,在run()中会阻塞当前线乘
    //如果是t.wait(),则表示是一个同步事件,在等待t中注册的时间后,即运行,不用调用io.run()
    io.run();
}
```



#### [Timer.1 同步](https://www.boost.org/doc/libs/1_74_0/doc/html/boost_asio/tutorial/tuttimer1/src.html) 

steady_timer是一个时间事件，io_context 是程序和电脑交互的接口,一般在ASIO的程序中,都要有一个io_service对象. 

这是一个同步事件，在执行  t.wait()时会等待，等待过程只要是因为steady_timer中注册了5秒的延迟。

```cpp
#include <iostream>
#include <boost/asio.hpp>

int main()
{
  boost::asio::io_context io;

  boost::asio::steady_timer t(io, boost::asio::chrono::seconds(5));
  t.wait();

  std::cout << "Hello, world!" << std::endl;

  return 0;
}
```

#### [Timer.2 异步](https://www.boost.org/doc/libs/1_74_0/doc/html/boost_asio/tutorial/tuttimer2/src.html)



异步事件，和同步只要区别在于，执行了steady_timer.async_wait()函数，而该函数回调了print函数，相当于给异步事件中注册了一个print函数，该函数并没有在async_wait中立即执行，或者在等待5秒后执行，是必须要执行  io.run()后才开始的。  io.run()会运行异步池中的事件。即当调用io.run后会在5秒后执行print函数



```cpp
#include <iostream>
#include <boost/asio.hpp>

void print(const boost::system::error_code& /*e*/)
{
  std::cout << "Hello, world!" << std::endl;
}

int main()
{
  boost::asio::io_context io;

  boost::asio::steady_timer t(io, boost::asio::chrono::seconds(5));
  t.async_wait(&print);

  io.run();

  return 0;
}
```



#### [Timer.3](https://www.boost.org/doc/libs/1_74_0/doc/html/boost_asio/tutorial/tuttimer3/src.html)

### Note

Boost.Asio允许同时使用异常处理或者错误代码， 所有的异步函数都有抛出错误和返回错误码两种方式的重载。 当函数抛出错误时， 它通常抛出
boost::system::system_error的错误。

看一下下面的代码片段：

```cpp
try {
	sock.connect(ep);
} catch(boost::system::system_error e) {
	std::cout << e.code() << std::endl;
}

//下面的代码片段和前面的是一样的：
boost::system::error_code err;
sock.connect(ep, err);
if ( err)
	std::cout << err << std::endl;
```

我们可以将语句放进try块中，也可以传入一个错误类型，当发生异常时，该类型会被修改写入，以下不在写try块，直接给函数中传入错误代码



```cpp
#include <iostream>
#include <boost/asio.hpp>
#include <boost/bind/bind.hpp>

void print(const boost::system::error_code& /*e*/,
    boost::asio::steady_timer* t, int* count)
{
  if (*count < 5)
  {
    std::cout << *count << std::endl;
    ++(*count);
    t->expires_at(t->expiry() + boost::asio::chrono::seconds(1));
    t->async_wait(boost::bind(print,
          boost::asio::placeholders::error, t, count));
     /*
  	 t->async_wait([t,count](auto er = boost::asio::placeholders::error()){
       print(er, t, count);
     });
  	*/
  }
}

int main()
{
  boost::asio::io_context io;
  int count = 0;
  boost::asio::steady_timer t(io, boost::asio::chrono::seconds(1));
  t.async_wait(boost::bind(print,
        boost::asio::placeholders::error, &t, &count));
  /*
   t.async_wait([&t,&count](auto er = boost::asio::placeholders::error()) {
        print(er, &t, &count);
    })
    在改成lambda一定要注意捕捉列表中的是引用，不能进行传值，因为print是要取地址的，不能传值，
    传值会用一个临时变量接受形参，会对临时对象取地址！！！！！
  */
  io.run();
  std::cout << "Final count is " << count << std::endl;
  return 0;
}
```

其中print参数有一个boost::system::error_code，但是这个参数并不是用户传的，而是在执行过程中，asio传的，所以需要一个placeholders。

因为async_wait接受一个可调用参数，所以需要进行绑定

上面代码使用的是bind进行绑定的，但是可以选择lambda，lambda中一定要注意指针的有效期，如在这个例子中的&t，可以将t包装为智能指针，这样就可以确保指针始终有效。

其中，expires_at可以重新设置steady_timer的过期时间（在该事件到达后执行），steady_timer.expiry()是事件的失效事件。

注意以下代码

```cpp
auto a1 = t->expiry();
std::this_thread::sleep_for(std::chrono::seconds(5));
auto a2 = t->expiry();  // a2和a1是一样的
```

```cpp
std::this_thread::sleep_for(std::chrono::seconds(5));
t->expires_at(t->expiry() + boost::asio::chrono::seconds(4));
t->async_wait([](auto er = boost::asio::placeholders::error()){
            std::cout<<1<<endl;
/*
两个执行效果基本一致的，虽然第一个的expires_at函数中加了4秒的延迟，第二个只有一秒的延迟
但是当执行std::this_thread::sleep_for(std::chrono::seconds(5));时，这个时候的t->expiry() + boost::asio::chrono::seconds(4)的值也不会大于执行t->expires_at（）语句的事件，所以该expires时间已经过去，所以会立即执行的。
*/
//**************************************************************************
std::this_thread::sleep_for(std::chrono::seconds(5));
t->expires_at(t->expiry() + boost::asio::chrono::seconds(1));
t->async_wait([](auto er = boost::asio::placeholders::error()){
            std::cout<<1<<endl;
```

#### [Timer.4](https://www.boost.org/doc/libs/1_73_0/doc/html/boost_asio/tutorial/tuttimer4/src.html)

```cpp
#include <iostream>
#include <boost/asio.hpp>
#include <boost/bind/bind.hpp>

class printer
{
public:
  printer(boost::asio::io_context& io)
    : timer_(io, boost::asio::chrono::seconds(1)),
      count_(0)
  {
    timer_.async_wait(boost::bind(&printer::print, this));
    //timer_.async_wait([this](auto er = boost::asio::placeholders::error()){this->print();});

  }
  ~printer()
  {
    std::cout << "Final count is " << count_ << std::endl;
  }
  void print()
  {
    if (count_ < 5)
    {
      std::cout << count_ << std::endl;
      ++count_;
      timer_.expires_at(timer_.expiry() + boost::asio::chrono::seconds(1));
      timer_.async_wait(boost::bind(&printer::print, this));
      //timer_.async_wait([this](auto er = boost::asio::placeholders::error()){this->print();});
 
    }
  }

private:
  boost::asio::steady_timer timer_;
  int count_;
};

int main()
{
  boost::asio::io_context io;
  printer p(io);
  io.run();

  return 0;
}
```



##   [Timer.5 - Synchronising handlers in multithreaded programs](https://www.boost.org/doc/libs/1_74_0/doc/html/boost_asio/tutorial/tuttimer5.html)

```cpp
class printer
{
public:
    /*strand类模板是一个执行器适配器，它保证对于那些通过它分派的处理程序，一个执行的处理程序将被允许在启动下一个处理程序之前完成。无论调用io_context::run()的线程有多少，这都是可以保证的。当然，处理程序仍然可以与其他未通过一个链分派或通过另一个链对象分派的处理程序并发执行。
    */
  printer(boost::asio::io_context& io)
    : strand_(boost::asio::make_strand(io)),
      timer1_(io, boost::asio::chrono::seconds(1)),
      timer2_(io, boost::asio::chrono::seconds(1)),
      count_(0)
  {/*
  当初始化异步操作时，每个回调处理程序被“绑定”到一个boost::asio::strand对象。函数的作用是::asio::bind_executor()返回一个新的处理程序，该处理程序通  过strand对象自动分派它所包含的处理程序。通过将处理程序绑定到同一串，我们确保了它们不能并发执行。
  */
    timer1_.async_wait(boost::asio::bind_executor(strand_,
          boost::bind(&printer::print1, this)));
	//timer1_.async_wait(boost::asio::bind_executor(strand_,[this](auto error = boost::asio::placeholders::error()){ this->print1();})); 
    //注意参数列表中的placeholders

    timer2_.async_wait(boost::asio::bind_executor(strand_,
          boost::bind(&printer::print2, this)));
  }

  ~printer()
  {
    std::cout << "Final count is " << count_ << std::endl;
  }
/*
在多线程程序中，如果异步操作的处理程序访问共享资源，它们应该是同步的。在本教程中，处理程序(print1和print2)使用的共享资源是std::cout和count_ data成员。
*/
  void print1()
  {
    if (count_ < 10)
    {
      std::cout << "Timer 1: " << count_ << std::endl;
      ++count_;

      timer1_.expires_at(timer1_.expiry() + boost::asio::chrono::seconds(1));

      timer1_.async_wait(boost::asio::bind_executor(strand_,
            boost::bind(&printer::print1, this)));
    }
  }

  void print2()
  {
    if (count_ < 10)
    {
      std::cout << "Timer 2: " << count_ << std::endl;
      ++count_;

      timer2_.expires_at(timer2_.expiry() + boost::asio::chrono::seconds(1));

      timer2_.async_wait(boost::asio::bind_executor(strand_,
            boost::bind(&printer::print2, this)));
    }
  }

private:
  boost::asio::strand<boost::asio::io_context::executor_type> strand_;
  boost::asio::steady_timer timer1_;
  boost::asio::steady_timer timer2_;
  int count_;
};
/*
main函数现在导致从两个线程调用io_context::run():主线程和另外一个线程。这是使用thread对象完成的。
就像对单个线程的调用一样，对io_context::run()的并发调用将在还有“工作”要做时继续执行。在所有异步操作完成之前，后台线程不会退出。
*/

int main()
{
  boost::asio::io_context io;
  printer p(io);
  boost::thread t(boost::bind(&boost::asio::io_context::run, &io));
  io.run();
  t.join();

  return 0;
}
```

strand类模板是一个执行器适配器，它保证对于那些通过它分派的处理程序，一个执行的处理程序将被允许在启动下一个处理程序之前完成。无论调用io_context::run()的线程有多少，这都是可以保证的。当然，处理程序仍然可以与其他未通过一个链分派或通过另一个链对象分派的处理程序并发执行。****



### [Daytime.1 - A synchronous TCP daytime client](https://www.boost.org/doc/libs/1_73_0/doc/html/boost_asio/tutorial/tutdaytime1.html)

```cpp
#include <iostream>
#include <boost/array.hpp>
#include <boost/asio.hpp>

using boost::asio::ip::tcp;

int main(int argc, char* argv[])
{
  try
  {
    if (argc != 2)
    {
      std::cerr << "Usage: client <host>" << std::endl;
      return 1;
    }

    boost::asio::io_context io_context;
	//我们需要将作为应用程序参数指定的服务器名称转换为TCP端点。 为此，我们使用ip::tcp::resolver对象。
    tcp::resolver resolver(io_context);
      
    //解析器采用主机名和服务名，并将它们转换为端点列表。 我们使用在argv[1]中指定的服务器名称,服务名称在在本例中为“daytime”执行解析调用。
	//使用ip::tcp::resolver::results_type类型的对象返回端点列表。该对象是一个范围，具有begin（）和end（）成员函数，可用于迭代结果。  
    tcp::resolver::results_type endpoints = resolver.resolve(argv[1], "daytime");

    tcp::socket socket(io_context);
    boost::asio::connect(socket, endpoints);
	//连接已打开。 我们现在要做的就是读取'daytime'的响应。

	//我们使用boost::array来保存接收到的数据。 boost::asio::buffer（）函数自动确定数组的大小，以帮助防止缓冲区溢出。 代替boost::array，我们可以使用char []或std::vector。
    for (;;)
    {
      boost::array<char, 128> buf;
      boost::system::error_code error;

      size_t len = socket.read_some(boost::asio::buffer(buf), error);
        
//当服务器关闭连接时，ip::tcp::socket::read_some（）函数将退出，并出现boost::asio::error::eof错误，这是我们知道退出循环的方式。
      if (error == boost::asio::error::eof)
        break; // Connection closed cleanly by peer.
      else if (error)
        throw boost::system::system_error(error); // Some other error.

      std::cout.write(buf.data(), len);
    }
  }
  catch (std::exception& e)
  {
    std::cerr << e.what() << std::endl;
  }

  return 0;
}
```

### [Daytime.2 - A synchronous TCP daytime server](https://www.boost.org/doc/libs/1_73_0/doc/html/boost_asio/tutorial/tutdaytime2.html)



```cpp
#include <ctime>
#include <iostream>
#include <string>
#include <boost/asio.hpp>

using boost::asio::ip::tcp;

std::string make_daytime_string()
{
  using namespace std; // For time_t, time and ctime;
  time_t now = time(0);
  return ctime(&now);
}

int main()
{
  try
  {
    boost::asio::io_context io_context;
	//需要创建一个ip::tcp::acceptor对象来侦听新的连接。 初始化为侦听TCP端口13（用于IP版本4）。
    tcp::acceptor acceptor(io_context, tcp::endpoint(tcp::v4(), 13));
	//这是一个迭代服务器，这意味着它将一次处理一个连接。 创建一个套接字，它将表示与客户端的连接，然后等待连接。
    for (;;)
    {
      tcp::socket socket(io_context);
      //同步异步区别只要在这里，如果是同步，在接收到一个socket前程序会一直等待，即只有当接收到socket时，才会执行std::string message = make_daytime_string();
      acceptor.accept(socket);
	  //客户正在访问我们的服务。 确定当前时间并将此信息传送给客户端。
      std::string message = make_daytime_string();

      boost::system::error_code ignored_error;
      boost::asio::write(socket, boost::asio::buffer(message), ignored_error);
    }
  }
  catch (std::exception& e)
  {
    std::cerr << e.what() << std::endl;
  }

  return 0;
}
```



### 	[Daytime.3 - An asynchronous TCP daytime server](https://www.boost.org/doc/libs/1_73_0/doc/html/boost_asio/tutorial/tutdaytime3.html)

```cpp
#include <ctime>
#include <iostream>
#include <string>
#include <boost/bind/bind.hpp>
#include <boost/shared_ptr.hpp>
#include <boost/enable_shared_from_this.hpp>
#include <boost/asio.hpp>

using boost::asio::ip::tcp;

std::string make_daytime_string()
{
    using namespace std; // For time_t, time and ctime;
    time_t now = time(0);
    return ctime(&now);
}
//我们将使用shared_ptr和enable_shared_from_this，因为只要有引用它的操作，我们就希望tcp_connection对象保持活动状态。
class tcp_connection: public boost::enable_shared_from_this<tcp_connection>
{
public:
    typedef boost::shared_ptr<tcp_connection> pointer;

    static pointer create(boost::asio::io_context& io_context)
    {
        return pointer(new tcp_connection(io_context));
    }

    tcp::socket& socket()
    {
        return socket_;
    }
//在函数start（）中，我们调用boost::asio::async_write（）将数据提供给客户端。 请注意，我们使用的是boost::asio::async_write（），而不是ip::tcp::socket::async_write_some（），以确保发送了整个数据块。
    void start()
    {//要发送的数据存储在类成员message_中，因为我们需要保持数据有效，直到异步操作完成。
        message_ = make_daytime_string();
	//启动异步操作时，如果使用boost::bind（），则必须仅指定与处理程序的参数列表匹配的参数。 在此程序中，两个参数占位符（boost::asio::placeholders::error和boost::asio::placeholders::bytes_transferred）可能已被删除，因为它们未在handle_write（）中使用。
        boost::asio::async_write(socket_, boost::asio::buffer(message_),
                                 boost::bind(&tcp_connection::handle_write, shared_from_this(),
                                             boost::asio::placeholders::error,
                                             boost::asio::placeholders::bytes_transferred));
        //现在，此客户端连接的所有其他操作都由handle_write（）负责。
    }

private:
    tcp_connection(boost::asio::io_context& io_context): socket_(io_context)
    {
    }
	//您可能已经注意到在handle_write（）函数的主体中未使用错误和bytes_transferred参数。 如果不需要参数，则可以将其从函数中删除，使其看起来像：
    void handle_write(const boost::system::error_code& /*error*/,
                      size_t /*bytes_transferred*/)
    {
    }

    tcp::socket socket_;
    std::string message_;
};

class tcp_server
{
public:
    //构造函数初始化一个接受器以侦听TCP端口13。
    tcp_server(boost::asio::io_context& io_context)
            : io_context_(io_context),
              acceptor_(io_context, tcp::endpoint(tcp::v4(), 13))
    {
        start_accept();
    }

private:
    //函数start_accept（）创建一个套接字并启动异步接受操作以等待新的连接。
    void start_accept()
    {
        tcp_connection::pointer new_connection = tcp_connection::create(io_context_);
        acceptor_.async_accept(new_connection->socket(),
                               boost::bind(&tcp_server::handle_accept, this, new_connection,
                                           boost::asio::placeholders::error));
    }
	//当start_accept（）启动的异步接受操作完成时，将调用handle_accept（）函数。 它为客户端请求提供服务，然后调用start_accept（）发起下一个接受操作。
    void handle_accept(tcp_connection::pointer new_connection,
                       const boost::system::error_code& error)
    {
        if (!error)
        {
            new_connection->start();
        }

        start_accept();
    }

    boost::asio::io_context& io_context_;
    tcp::acceptor acceptor_;
};

int main()
{
    try
    {//我们需要创建一个服务器对象来接受传入的客户端连接。 io_context对象提供服务器对象将使用的I / O服务，例如套接字。
        boost::asio::io_context io_context;
        tcp_server server(io_context);
        // Run the io_context object so that it will perform asynchronous operations on your behalf. 
        io_context.run();
    }
    catch (std::exception& e)
    {
        std::cerr << e.what() << std::endl;
    }

    return 0;
}
```

上面是TCP，下面是UDP两个链接协议差别很大。只要是TCP是端到端的，是面向安全的链接，在链接开始必须进行三次握手，断开进行四次握手确认。这样保证了数据可靠性，即是有了丢包现象也会重发。一对一的可靠链接。

UPD并不进行链接检查，直接发送数据，而且可以群发，广播。UDP并不会对ip数据进行较多的处理，仅仅加了udp头信息仅转发出去，且并不需要链接的。在事实性应用广泛，在安全完整性上是极大的缺点。UDP可以做视频通话，例如卡顿现象，因为服务器并不在乎接收端的网络状态，只管自己发送数据。而TCP可以应用于传输文件，因为传输文件必须确保文件的完整性。



### [Daytime.4 - A synchronous UDP daytime client](https://www.boost.org/doc/libs/1_74_0/doc/html/boost_asio/tutorial/tutdaytime4.html)

```cpp
using boost::asio::ip::udp;

int main(int argc, char* argv[])
{
  try
  {
    if (argc != 2)
    {
      std::cerr << "Usage: client <host>" << std::endl;
      return 1;
    }

    boost::asio::io_context io_context;
	//我们使用ip :: udp :: resolver对象根据主机名和服务名找到要使用的正确远程端点。 ip :: udp :: v4（）参数将查询限制为仅返回IPv4端点。
    udp::resolver resolver(io_context);
    udp::endpoint receiver_endpoint =
      *resolver.resolve(udp::v4(), argv[1], "daytime").begin();

    //保证ip :: udp :: resolver :: resolve（）函数在不失败的情况下至少返回列表中的一个端点。 这意味着直接取消引用返回值是安全的。
    udp::socket socket(io_context);
    socket.open(udp::v4());  //UDP 和TCP最大的差别就是这里不是connect，而是open
      
    // 实际上可以将上面改为 udp::socket socket(io_context, udp::v4());
    boost::array<char, 1> send_buf  = { '0' };
      std::string s = "123";
      socket.send_to(boost::asio::buffer(s), receiver_endpoint);
      boost::array<char, 128> recv_buf;
      udp::endpoint sender_endpoint;
      //由于UDP是面向数据报的，因此我们将不使用流套接字。 创建一个ip :: udp :: socket并启动与远程端点的联系。
      //现在，我们需要准备好接受服务器发回给我们的任何内容。 接收服务器响应的端点将通过ip :: udp :: socket :: receive_from（）进行初始化。
      
	  udp::endpoint remote_endpoint;  

      size_t len = socket.receive_from(boost::asio::buffer(recv_buf), remote_endpoint);

    std::cout.write(recv_buf.data(), len);
  }
  catch (std::exception& e)
  {
    std::cerr << e.what() << std::endl;
  }

  return 0;
}
```



### [Daytime.5 - A synchronous UDP daytime server](https://www.boost.org/doc/libs/1_74_0/doc/html/boost_asio/tutorial/tutdaytime5.html)

```cpp
#include <ctime>
#include <iostream>
#include <string>
#include <boost/array.hpp>
#include <boost/asio.hpp>

using boost::asio::ip::udp;

std::string make_daytime_string()
{
  using namespace std; // For time_t, time and ctime;
  time_t now = time(0);
  return ctime(&now);
}

int main()
{
  try
  {
    boost::asio::io_context io_context;
    udp::socket socket(io_context, udp::endpoint(udp::v4(), 13));
//等待客户开始与我们联系。 remote_endpoint对象将由ip :: udp :: socket :: receive_from（）填充。
 // TCP有tcp::acceptor acceptor(...)，因为tcp面向连接

      for (;;)
    {
      boost::array<char, 1> recv_buf;
      udp::endpoint remote_endpoint;
      socket.receive_from(boost::asio::buffer(recv_buf), remote_endpoint);
//确定我们要发回给客户的东西。
      std::string message = make_daytime_string();
//将响应发送到remote_endpoint。
      boost::system::error_code ignored_error;
      socket.send_to(boost::asio::buffer(message),
          remote_endpoint, 0, ignored_error);
    }
  }
  catch (std::exception& e)
  {
    std::cerr << e.what() << std::endl;
  }

  return 0;
}
```

client中的socket.send_to(boost::asio::buffer(s), receiver_endpoint); 和server中的socket.receive_from(boost::asio::buffer(recv_buf), remote_endpoint);是什么？为什么无法打印出来

## [ An asynchronous UDP daytime server](https://www.boost.org/doc/libs/1_73_0/doc/html/boost_asio/tutorial/tutdaytime6.html)

```cpp
#include <ctime>
#include <iostream>
#include <string>
#include <boost/array.hpp>
#include <boost/bind/bind.hpp>
#include <boost/shared_ptr.hpp>
#include <boost/asio.hpp>

using boost::asio::ip::udp;

std::string make_daytime_string()
{
  using namespace std; // For time_t, time and ctime;
  time_t now = time(0);
  return ctime(&now);
}

class udp_server
{
public:
  udp_server(boost::asio::io_context& io_context)
    : socket_(io_context, udp::endpoint(udp::v4(), 13))
  {
    start_receive();
  }

private:
  void start_receive()
  {
    socket_.async_receive_from(
        boost::asio::buffer(recv_buffer_), remote_endpoint_,
        boost::bind(&udp_server::handle_receive, this,
          boost::asio::placeholders::error,
          boost::asio::placeholders::bytes_transferred));
  }

  void handle_receive(const boost::system::error_code& error,
      std::size_t /*bytes_transferred*/)
  {
    if (!error)
    {
      boost::shared_ptr<std::string> message(
          new std::string(make_daytime_string()));

      socket_.async_send_to(boost::asio::buffer(*message), remote_endpoint_,
          boost::bind(&udp_server::handle_send, this, message,
            boost::asio::placeholders::error,
            boost::asio::placeholders::bytes_transferred));

      start_receive();
    }
  }

  void handle_send(boost::shared_ptr<std::string> /*message*/,
      const boost::system::error_code& /*error*/,
      std::size_t /*bytes_transferred*/)
  {
  }

  udp::socket socket_;
  udp::endpoint remote_endpoint_;
  boost::array<char, 1> recv_buffer_;
};

int main()
{
  try
  {
    boost::asio::io_context io_context;
    udp_server server(io_context);
    io_context.run();
  }
  catch (std::exception& e)
  {
    std::cerr << e.what() << std::endl;
  }

  return 0;
}
```





### buffers/reference_counted

这个清单中async_write的buffer为shared_const_buffer，该shared_const_buffer类将原来的char * 包装为boost::asio::const_buffer，并重写begin（），和end（）方法。

实际上，boost::asio::const_buffer包装了原始数据，因为这是写入，所以buffer是const的，可以声明为const

```cpp
#include <boost/asio.hpp>
#include <iostream>
#include <memory>
#include <utility>
#include <vector>
#include <ctime>

using boost::asio::ip::tcp;

// A reference-counted non-modifiable buffer class.
class shared_const_buffer
{
public:
  // Construct from a std::string.
  explicit shared_const_buffer(const std::string& data)
    : data_(new std::vector<char>(data.begin(), data.end())),
      buffer_(boost::asio::buffer(*data_))
  {
  }

  // Implement the ConstBufferSequence requirements.
  const boost::asio::const_buffer* begin() const { return &buffer_; }
  const boost::asio::const_buffer* end() const { return &buffer_ + 1; }

private:
  std::shared_ptr<std::vector<char> > data_;
  boost::asio::const_buffer buffer_;
};

class session
  : public std::enable_shared_from_this<session>
{
public:
  session(tcp::socket socket)
    : socket_(std::move(socket))
  {
  }

  void start()
  {
    do_write();
  }

private:
  void do_write()
  {
    std::time_t now = std::time(0);
    shared_const_buffer buffer(std::ctime(&now));

    auto self(shared_from_this());
    boost::asio::async_write(socket_, buffer,
        [self](boost::system::error_code /*ec*/, std::size_t /*length*/)
        {
        });
  }

  // The socket used to communicate with the client.
  tcp::socket socket_;
};

class server
{
public:
  server(boost::asio::io_context& io_context, short port)
    : acceptor_(io_context, tcp::endpoint(tcp::v4(), port))
  {
    do_accept();
  }

private:
  void do_accept()
  {
    acceptor_.async_accept(
        [this](boost::system::error_code ec, tcp::socket socket)
        {
          if (!ec)
          {
            std::make_shared<session>(std::move(socket))->start();
          }

          do_accept();
        });
  }

  tcp::acceptor acceptor_;
};

int main(int argc, char* argv[])
{
  try
  {
    if (argc != 2)
    {
      std::cerr << "Usage: reference_counted <port>\n";
      return 1;
    }

    boost::asio::io_context io_context;

    server s(io_context, std::atoi(argv[1]));

    io_context.run();
  }
  catch (std::exception& e)
  {
    std::cerr << "Exception: " << e.what() << "\n";
  }

  return 0;
}
```

实际上可以将do_write改为以下，这样就不需要自定义shared_const_buffer类了。

```cpp
  void do_write()
  {
    std::time_t now = std::time(0);
    auto time = std::ctime(&now);

    auto self(shared_from_this());
   
    auto asd = boost::asio::const_buffer(time, std::strlen(time));
    boost::asio::async_write(socket_, asd,
        [self](boost::system::error_code /*ec*/, std::size_t /*length*/)
        {
        });
  }
```



ASIO 中的buffer只有两种，即mutable_buffer和const_buffer，都支持


mutable_buffer buffer ( void * data, std::size_t size_in_bytes);
const_buffer buffer (const  void * data, std::size_t size_in_bytes);
实际上可以直接根据data是否是底层const指针自动判断类型。





TCP 有read，write

UDP和icmp 是send和receive

 // TCP的tcp::acceptor acceptor(...)，实际上就是进行握手操作，而UDP和icmp 不需要

实际上socket的local_endpoint都是隐式绑定的，只有当connect的时候才会有remote_endpoint

因为UDP，ICMP都是面向无连接的所以使用connect也没有remote_endpoint,但是却可以隐式指定目标端口，就不需要进行send_to , 可以使用send

实际上receive_from（boost::asio::buffer(), remote_endpoint)) 会将发送端的endpoint信息写入

remote_endpoint中。

在asynchronous UDP client和server中为什么是client先给server发数据然后再接受，因为只有发送过去了，server才可以使用receive_from来记录client的endpoint，然后再将真实数据发送给client。 