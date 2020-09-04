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



 [Timer.5 - Synchronising handlers in multithreaded programs](https://www.boost.org/doc/libs/1_74_0/doc/html/boost_asio/tutorial/tuttimer5.html)

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

