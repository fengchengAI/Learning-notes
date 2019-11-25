### ASIO







以上

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