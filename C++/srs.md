在srs_main_servicer.cpp中有run_master(SrsServer* svr)

```c++
// main/srs_main_server.cpp:
srs_error_t run_master(SrsServer* svr)
{    
    if ((err = svr->listen()) != srs_success) {
        return srs_error_wrap(err, "listen");
    }
}
...
srs_error_t SrsServer::listen()
{    
    if ((err = listen_rtmp()) != srs_success) {
        return srs_error_wrap(err, "rtmp listen");
    }
}
...
srs_error_t SrsServer::listen_rtmp()
{
    SrsListener* listener = new SrsBufferListener(this, SrsListenerRtmpStream);
    if ((err = listener->listen(ip, port)) != srs_success) {
        srs_error_wrap(err, "rtmp listen %s:%d", ip.c_str(), port);
    }
}
...

srs_error_t SrsBufferListener::listen(string i, int p)
{
    listener = new SrsTcpListener(this, ip, port);
    if ((err = listener->listen()) != srs_success) {
        return srs_error_wrap(err, "buffered tcp listen");
    }
}
...
srs_error_t SrsTcpListener::listen()
{
   //srs_tcp_listen 并不是成员函数
    if ((err = srs_tcp_listen(ip, port, &lfd)) != srs_success) {
        return srs_error_wrap(err, "listen at %s:%d", ip.c_str(), port);
    }
    // srs_tcp_listen会用do_srs_tcp_listen创建一个socket，并进行bind,listen，并进行了一定的设置
    // 在_st_netfd_new(osfd, 1, 1)中用st.common.h中的_st_netfd_t进行了包装，即
    // 最终创建后返回给SrsTcpListener::lfd
}
    // 创建完socket后创建协程，将SrsTcpListener将作为hander赋值给SrsSTCoroutine中的		ISrsCoroutineHandler* handler
    trd = new SrsSTCoroutine("tcp", this); 
    if ((err = trd->start()) != srs_success) {
        return srs_error_wrap(err, "start coroutine");
    }    
}
...
// 创建完socket后创建协程
srs_error_t SrsSTCoroutine::start()
{
    // _pfn_st_thread_create 这里是协程，即State Threads
    // _pfn_st_thread_create会将pfn进行包装为_st_thread_t
    if ((trd = (srs_thread_t)_pfn_st_thread_create(pfn, this, 1, 0)) == NULL) {
        err = srs_error_new(ERROR_ST_CREATE_CYCLE_THREAD, "create failed");
        srs_freep(trd_err);
        trd_err = srs_error_copy(err);
        return err;
    }
    started = true;
    return err;
}
...
//其中pfn为
    void* SrsSTCoroutine::pfn(void* arg)
{
    SrsSTCoroutine* p = (SrsSTCoroutine*)arg;
    srs_error_t err = p->cycle();
...
}
//实际上这里将p->cycle()调用给State Threads管理了。
//当p->cycle()发生时。会调用handler->cycle();
srs_error_t SrsSTCoroutine::cycle()
{ 
    if (_srs_context) {
        if (context) { //构造函数中没有传入这个，应该位0
            _srs_context->set_id(context);
        } else {
            context = _srs_context->generate_id();
        }
    }
    srs_error_t err = handler->cycle();
    if (err != srs_success) {
        return srs_error_wrap(err, "coroutine cycle");
    }
    cycle_done = true;
    return err;
}
...
// 而hander是SrsTcpListener，于是进入
    srs_error_t SrsTcpListener::cycle()
{
    srs_error_t err = srs_success;
    
    while (true) {
        if ((err = trd->pull()) != srs_success) {
            return srs_error_wrap(err, "tcp listener");
        }
        
        srs_netfd_t fd = srs_accept(lfd, NULL, NULL, SRS_UTIME_NO_TIMEOUT);
        if(fd == NULL){
            return srs_error_new(ERROR_SOCKET_ACCEPT, "accept at fd=%d", srs_netfd_fileno(lfd));
        }
        
	    if ((err = srs_fd_closeexec(srs_netfd_fileno(fd))) != srs_success) {
	        return srs_error_wrap(err, "set closeexec");
	    }
        
        if ((err = handler->on_tcp_client(fd)) != srs_success) {
            return srs_error_wrap(err, "handle fd=%d", srs_netfd_fileno(fd));
        }
    }
    
    return err;
}
//这里主循环了，先监听srs_accept，然后将连接到的socket包装为srs_netfd_t，送入handler->on_tcp_client(fd)
// 此时handler为SrsBufferListener

srs_error_t SrsBufferListener::on_tcp_client(srs_netfd_t stfd)
{   // 这个stfd是accept回传的
    // SrsBufferListener没有server，这是父类的
    srs_error_t err = server->accept_client(type, stfd);  // 指向SrsServer
    return srs_success;
}
...
 
srs_error_t SrsServer::accept_client(SrsListenerType type, srs_netfd_t stfd)
{   
    SrsConnection* conn = NULL;
    if ((err = fd2conn(type, stfd, &conn)) != srs_success) {
        if (srs_error_code(err) == ERROR_SOCKET_GET_PEER_IP && _srs_config->empty_ip_ok()) {
            srs_close_stfd(stfd); srs_error_reset(err);
            return srs_success;
        }
        return srs_error_wrap(err, "fd2conn");
    }
    // directly enqueue, the cycle thread will remove the client.
    conns.push_back(conn);
    // cycle will start process thread and when finished remove the client.
    // @remark never use the conn, for it maybe destroyed.
    if ((err = conn->start()) != srs_success) {
        return srs_error_wrap(err, "start conn coroutine");
    }
    return err;
}
// 此时的conn是fd2conn返回的一个rtmp服务了
// 且conn->start()也就是SrsConnection::start(),因为SrsRtmpConn并没有重写start方法
// 所以是调用父类的
    srs_error_t SrsConnection::start()
{
    srs_error_t err = srs_success;
    
    if ((err = skt->initialize(stfd)) != srs_success) {
        return srs_error_wrap(err, "init socket");
    }
    if ((err = trd->start()) != srs_success) {  // 这里为accept接受到的socket创建一个协程
        return srs_error_wrap(err, "coroutine");
    }
    
    return err;
}

//trd->start()会调用SrsSTCoroutine::start()
srs_error_t SrsSTCoroutine::start()
{
    srs_error_t err = srs_success;
    
    if (started || disposed) {
        if (disposed) {
            err = srs_error_new(ERROR_THREAD_DISPOSED, "disposed");
        } else {
            err = srs_error_new(ERROR_THREAD_STARTED, "started");
        }

        if (trd_err == srs_success) {
            trd_err = srs_error_copy(err);
        }
        
        return err;
    }
    // _pfn_st_thread_create 这里是协程，即State Threads
    if ((trd = (srs_thread_t)_pfn_st_thread_create(pfn, this, 1, 0)) == NULL) {
        err = srs_error_new(ERROR_ST_CREATE_CYCLE_THREAD, "create failed");
        
        srs_freep(trd_err);
        trd_err = srs_error_copy(err);
        
        return err;
    }
    
    started = true;

    return err;
}
// 此时的注册给SrsSTCoroutine的是SrsConnection，最后会调用SrsConnection.cycle()
srs_error_t SrsConnection::cycle()
{
    srs_error_t err = do_cycle();  // 这里进入SrsRtmpConn::do_cycle()
    
    // Notify manager to remove it.
    manager->remove(this);
    
    // success.
    if (err == srs_success) {
        srs_trace("client finished.");
        return err;
    }

    return srs_success;
}

```





先进行receive

在SrsRtmpConn::publishing(SrsSource* source)前就完成了rtmp的握手和连接，此时客户端已经开始发送消息了

```c++
srs_error_t SrsRtmpConn::publishing(SrsSource* source)
{
    srs_error_t err = srs_success;
    
    if ((err = acquire_publish(source)) == srs_success) {

        SrsPublishRecvThread rtrd(rtmp, req, srs_netfd_fileno(stfd), 0, this, source, _srs_context->get_id());
        err = do_publishing(source, &rtrd);  //SrsRtmpConn::do_publishing
        rtrd.stop();
    }
    return err;
}
```

acquire_publish进行了一些初始化工作

```c++
srs_error_t SrsRtmpConn::do_publishing(SrsSource* source, SrsPublishRecvThread* rtrd)
{
    // start isolate recv thread.
    /* 构建并启动一个专门用于接收客户端推流数据的 recv 线程 */
    // rtrd为SrsPublishRecvThreadSrsPublishRecvThread
    if ((err = rtrd->start()) != srs_success) {  // SrsPublishRecvThread::start()
        return srs_error_wrap(err, "rtmp: receive thread");
    }
    return err;
}

```

```c++
srs_error_t SrsPublishRecvThread::start()
{
    srs_error_t err = srs_success;
    //trd 指向SrsRecvThread
    if ((err = trd.start()) != srs_success) { //SrsRecvThread::start()
        err = srs_error_wrap(err, "publish recv thread");
    }
    
    ncid = cid = trd.cid();

    return err;
}
```

```c++
srs_error_t SrsRecvThread::start()
{
    srs_error_t err = srs_success;
    
    srs_freep(trd);
    trd = new SrsSTCoroutine("recv", this, _parent_cid);
    //创建协程执行SrsRecvThread.cycle()
    if ((err = trd->start()) != srs_success) {
        return srs_error_wrap(err, "recv thread");
    }
    
    return err;
}
```

```c++
srs_error_t SrsRecvThread::cycle()
{
    srs_error_t err = srs_success;
   
    rtmp->set_recv_timeout(SRS_UTIME_NO_TIMEOUT);
    
    pumper->on_start();
    
    if ((err = do_cycle()) != srs_success) {  // SrsRecvThread::do_cycle()
        err = srs_error_wrap(err, "recv thread");
    }
    
    // reset the timeout to pulse mode.
    rtmp->set_recv_timeout(timeout);
    
    pumper->on_stop();
    
    return err;
}
```

```c++
srs_error_t SrsRecvThread::do_cycle()
{
    srs_error_t err = srs_success;
    
    while (true) {
        if ((err = trd->pull()) != srs_success) {
            return srs_error_wrap(err, "recv thread");
        }
        
        // When the pumper is interrupted, wait then retry.
        if (pumper->interrupted()) {
            srs_usleep(timeout);
            continue;
        }
        
        SrsCommonMessage* msg = NULL;
        
        // Process the received message.
        if ((err = rtmp->recv_message(&msg)) == srs_success) {
            err = pumper->consume(msg);
        }
        
        if (err != srs_success) {
            // Interrupt the receive thread for any error.
            trd->interrupt();
            
            // Notify the pumper to quit for error.
            pumper->interrupt(err);
            
            return srs_error_wrap(err, "recv thread");
        }
    }
    
    return err;
}
```

