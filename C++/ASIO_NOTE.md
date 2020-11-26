
```cpp
// synchronous TCP daytime client
tcp::resolver resolver(io_context);
tcp::resolver::results_type endpoints = resolver.resolve(argv[1], "daytime");

tcp::socket socket(io_context);
boost::asio::connect(socket, endpoints);


// tcp::socket socket(io_context, endpoints->endpoint()); 如果改为现在的，就会报错
//bind: Address already in use

// 但是事实上endpoints只有一个就是127.0.0.1, 所以并不存在多个，为什么不直接绑定呢
  for (;;)
    {
      boost::array<char, 128> buf;
      boost::system::error_code error;
//local_endpoint ：127.0.0.1：44264

//remote_endpoint：127.0.0.1：13


      size_t len = socket.read_some(boost::asio::buffer(buf), error);
```

```cpp
//// synchronous TCP daytime server
tcp::acceptor acceptor(io_context, tcp::endpoint(tcp::v4(), 13));
for (;;)
{
  tcp::socket socket(io_context);

  acceptor.accept(socket);
  std::string message = make_daytime_string();

  boost::system::error_code ignored_error;
  boost::asio::write(socket, boost::asio::buffer(message), ignored_error);
}
```




```cpp
// synchronous UDP daytime client
udp::resolver resolver(io_context);
udp::endpoint receiver_endpoint =
      *resolver.resolve(udp::v4(), argv[1], "daytime").begin();//v4（）参数仅返回IPv4端点

udp::socket socket(io_context);
socket.open(udp::v4());
boost::array<char, 1> send_buf  = { '0' };
std::string s = "123";
socket.send_to(boost::asio::buffer(s), receiver_endpoint);
boost::array<char, 128> recv_buf;
udp::endpoint sender_endpoint;

size_t len = socket.receive_from(
boost::asio::buffer(recv_buf), sender_endpoint);
```
 ```CPP
// synchronous UDP daytime server
udp::socket socket(io_context, udp::endpoint(udp::v4(), 13));
//等待客户开始与我们联系。 remote_endpoint对象将由ip :: udp :: socket :: receive_from（）填充。
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
 ```

 

 