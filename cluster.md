## 概述
* 本文主要是把集群的线程（服务）和通信机制讲述清楚，所有的设计都是基于原有的信长C++引擎来实现的
    * [新增线程](#1)
    * [线程启动](#2)
        * [创建&初始化](#2)
    * [新增服务](#3)
    * [服务启动](#4)
        * [创建&初始化](#4)
    * [集群客户端与服务器建立连接](#6)
    * [发起远程异步调用](#7)
        * [使用场景说明](#20)
        * [网络消息包格式说明](#21)
        * [调用方式说明](#22)
	    * [发送方](#11)
	    * [接收方](#12)
    * [错误处理](#8)
    * [热更新节点配置文件](#9)
## <i id="1">新增线程</i>
* 新增的线程只有一个ClusterMgr
    * 线程继承关系ClusterMgr->EvThread->ThreadBase，类的主要成员如下图 
	* ![ClusterMgr类成员+函数](https://user-images.githubusercontent.com/50430941/133437078-f8734c63-31b3-414f-91c5-895b0128cfc0.png)
    * ThreadBase是线程基类，只有一个重要的成员变量：
        * _thread_：线程实例，创建一个新的线程，在这个线程内跑传进来的函数。
    * EvThread类是使用了libevent库的线程基类
        * 3个重要的成员函数
            * _L_：被线程直接使用的lua虚拟机，线程通过lua虚拟机发消息给其它线程，比如线程收到消息包，会通过lua虚拟机调用C库函数发给其它服务。
            * _evpairs_：其它线程通过_evpairs_的writeData函数往_wBev_(是EvPairs的一个数据包写入缓冲池)写入数据，ClusterMgr线程则通过_evpairs_的readData函数从_rBev_(是EvPairs的一个数据包接收缓冲池)读消息。
            * _base_：libevent的核心数据结构（CluseterMgr线程内部独立维护一个event_base实例，线程内共享）。
        * 接口
            * EvThread：构造函数，初始化数据
            * runThread：线程启动函数
            * sendMsgData：往_evpairs_写入数据
            * initThreadOpenLib：初始化C库函数
            * loadMainFile：加载lua文件到lua虚拟机
    * ClusterMgr是集群通信线程类
        * ClusterMgr：
            * 字段
                * 内部有两个结构，这两个类都继承TcpObject，这两个类是集群通信的客户端（TcpClient）和服务端（TcpServer）类。
            * 接口
                * ClusterMgr：构造函数
                * start：线程启动入口
                * run：线程内loop的函数
                * onAccpeted：连接事件回调
                * onOpened：加入event事件列表
                * tryConnect：主动发起连接
                * onConnectCB：连接请求回调函数
        * TcpObject：
            * 字段
                * _handle_：处理业务的服务句柄
                * _sessionDict_：已建立连接的session字典
                * _totalSessionId_：用来生成唯一id
                * _base_：libevent的核心数据结构（CluseterMgr线程内部独立维护一个event_base实例，线程内共享）。
            * 接口
                * getHandle：获取服务句柄
                * createSession：创建新的session实例，并加入sessionDict
                * removeSession：从sessionDict中移除无效的session
                * errorCB：session抛出的错误接收函数，准备移除session
        * TcpServer：
            * 字段
                * _ip_：端口监听的服务器ip
                * _port_：端口监听的端口
                * _fd_：端口监听的套接字
                * _event_：libevent的事件监听结构体
            * 接口
                * initListen：设置TCP端口监听的参数
                * startListen：libevent开始监听TCP端口
                * onAccpeted：新连接事件回调
                * onOpened：接收客户端的连接
        * TcpClient
            * 字段：不用额外加字段，继承TcpObject的字段足够使用
            * 接口
                * tryConnect：主动发起连接
                * onConnectCB：连接请求回调函数
                * disconnect：客户端主动断开连接
        * Session：
            * 字段
                * _sessionId_：session实例唯一id，是sessionDict的key
                * _fd_：tcp连接的套接字
                * _ip_：对端服务器ip
                * _bev_：bufferevent的读写缓存
                * _handle_：处理session业务的服务句柄(session收到数据包后，会把数据包转发给_handle_对应的服务)
                * _readBuff_：收到的数据包，会先放入这个缓存，收齐后，抛给业务层
                * _readStatus_：标记正在读包头还时数据
                * _readSize_：还需要读几个字节，和_readStatus_配套使用
                * _buffIdx_：标记_readBuff_的索引
                * _active_：标记session是否可用
                * _read_cb_：读到一个完整包后的回调
                * _error_cb_：错误回调
            * 接口
                * init：初始化数据
                * destroy：销毁数据
                * sendData：发送数据
                * readCB：读取数据，转发给服务
                * errorCB：错误回调，注册到event_base的回调函数
    * ClusterMgr线程主要是实现集群间的通信
        * 基本逻辑是worker线程中的服务（调用者）需要发起远端异步调用时，向ClusterMgr线程发送消息，ClusterMgr收到消息后，连接服务端，然后把远端异步调用请求发给服务端。服务端的ClusterMgr线程收到请求后，根据请求参数把远端异步调用请求转发给被调用者，被调用者处理完请求后，原路把结果返回给调用者。
    * 其它线程查看参考文档[信长C++引擎文档](#10)的 **线程模型**
## <b id="2">线程启动</b>
* 在日志线程启动完后，会启动一条ClusterMgr线程。
* ![ClusterMgr内存模型](https://user-images.githubusercontent.com/50430941/133437320-0a8519b8-4bf5-4a6b-9d34-0f8a486f27d9.png)
    * 线程构造函数中，会调用基类构造函数，初始化类的成员数据。
    * ClusterMgr线程start函数中，调用基类的EvThread::runThread函数
        * 首先会初始化lua虚拟机的环境，把一些通用的C库函数库函数和线程内专用的C库函数都注册到lua虚拟机，然后把lua脚本加载到lua虚拟机，调用lua的__main__函数
        * 接着执行_evpaires_的初始化，调用EvPairs::initPairs函数，设置好_evpairs_的回调函数和监听事件。_evpairs_内部维护一个libevnet的读写缓存，负责处理与内核的数据读写。
        * ![evpaires实例](https://user-images.githubusercontent.com/50430941/133437369-251d43b2-ef40-4376-98f9-88f0dae07d21.png)
        * 最后，创建线程，在线程内跑CluaterMgr::run这个函数
* 其它线程的启动可以查看参考文档[信长C++引擎文档](#10)的 **线程启动机制**
## <b id="3">新增服务</b>
* 新增的服务有两个
    * clusterserver：集群通信的服务端消息处理服务
        * 字段
            * session2context：保存远程调用的数据
            * clusterName：存储服务器的ip和port
        * 接口
            * __main__：主函数，虚拟机运行lua脚本时，第一个调用的函数，加载配置文件
            * _msg_dispatch_：服务发送过来的消息入口
            * _recv_data_：ClusterMgr线程发送过来的消息入口
            * callback：远程调用回调函数
            * doAfterInit：通知ClusterMgr监听端口（作为集群服务器）
            * onInitListened：ClusterMgr线程初始化listen完成通知
            * startListen：开始作为集群服务器的端口监听
            * onDisconnect：连接断开连接事件
    * clusterclient：集群通信的客户端消息处理服务
        * 字段
            * session2context：保存远程调用的数据
            * requestCache：连接未建立时的请求缓存
            * clusterName：存储服务器的ip和port
            * connectStatusDict：服务器连接状态
        * 接口
            * __main__：主函数，虚拟机运行lua脚本时，第一个调用的函数，加载配置文件
            * remoteSend：远程send函数
            * remoteCall：远程call函数
            * _msg_dispatch_：服务发送过来的消息入口
            * _recv_data_：ClusterMgr线程发送过来的消息入口
            * callback：远程调用回调函数
            * onDisconnect：连接断开连接事件
    * 新增服务用于处理远端异步调用，clientservice发起远端异步调用，由客户端的ClusterMgr线程发给服务端的ClusterMgr线程，服务端的ClusterMgr线程转发给clientservice服务，clientservice转发给具体的业务处理完后
    * remoteCall原路把处理结果返回
    * ![服务发起call调用](https://user-images.githubusercontent.com/50430941/133437471-98948fec-b8b7-4d40-9063-cba916b0f5c5.png)
    * remoteSend不需要返回
    * ![服务send消息到远程程服务器](https://user-images.githubusercontent.com/50430941/133437508-4b5c8cf7-e654-4cc3-a792-fe9723c0337f.png)
## <b id="4">服务启动</b>
* 配置说明
    * 服务器的启动命令示例(以后可能会修改启动命令，但是nodename参数肯定会有)：nohup ./evpt nodename &
        * nodename：当前节点的名字，如battle_1
    * 新加一个服务器的配置文件：clustername.lua
    * 服务器之间的关系是peer to peer
        * 每个节点有一个独一无二的nodename，所有服务器都可以在nodename.lua中找到属于自己的配置
        ```
            battle_1    =   "192.168.10.1:8001"
            game_1      =   "192.168.10.1:8002"
        ```
* cluster类（clusterserver、clusterclient）服务
    * 由lanucher服务启动cluster类服务
        * launcher服务调用创建snlua类型服务的接口
        * 具体的创建服务流程请看[信长C++引擎文档](#10)的 **Actor模式在GameSvr中的运用**
    * 在服务的创建过程中，会调用服务的__main__函数，函数会读取clustername.lua（格式：nodeName=ip:port）配置文件，内存中把服务器的地址加载到虚拟机中。
        * clusterName数据结构：{nodeName = {ip=string, port=int}}
    * clusterclient服务启动后自加载配置，以及用配置初始化connectStatusDict，格式connectStatusDict={nodeName=disconnect|connecting|connected}，这个字典记录了与其它节点的连接状态
    * clusterserver服务会在doAfterInit（初始化完成后）中发消息给ClusterMgr，监听配置在clustername.lua中自己ip所对应的端口。
    * ![server端口监听](https://user-images.githubusercontent.com/50430941/133437545-b21c5814-fdde-4d39-996e-d1abbce396bd.png)
* 补充说明：现在的设计是clusterserver服务通知ClusterMgr线程开启监听。如果由ClusterMgr线程自己去开启监听，由于ClusterMgr线程比clusterserver服务先启动，当ClusterMgr线程收到连接，并建立连接后，开始收数据包，这时发接收到的消息包给clusterserver进行处理，clusterserver服务可能还没创建或者没初始化完成，导致消息传递中断了，服务器会出现各种奇怪的错误，增加业务处理的复杂性。
## <b id="6">集群客户端与服务器建立连接</b>
* 第一次往某个服务端发起远程调用请求时，客户端内部：
    * connectStatusDict中与服务端的连接状态为disconnect，请求会放到requestCache中。clusterclient服务会发消息给ClusterMgr线程，connectStatusDict中与服务端的连接状态转变为connecting，ClusterMgr线程开始连接服务端。连接成功建立后，connectStatusDict中与服务端的连接状态变为connected，把requestCache中的请求发给服务端，并把requestCache中的与这个服务端相关的请求生成唯一session后，转移到session2context中。
* 下图补充说明：已用红字标明先后步骤，其中蓝箭头是发起连接的步骤，黑箭头是服务器监听的步骤，红箭头是客户端连接成功的步骤。
* ![新建连接](https://user-images.githubusercontent.com/50430941/133437608-0b68edda-dbf6-4318-906a-b66350a40c7e.png)
## <b id="7">发起远程异步调用</b>
* <b id="20">使用场景说明</b>
    * remoteCall是用于一个节点的服务调用另一个节点的某个服务，因为目前框架无法挂起，所以被调用者的服务==不能再向其它服务或者节点发起调用==
    * 使用示例
        * 调用者
            * ![调用示例](https://user-images.githubusercontent.com/50430941/133437663-68a666be-4c5c-473b-8584-f203eadf7d90.png)
    * 补充：如果被调用者需要向其它服务或节点发起请求，则只能使用remoteSend
        * remoteSend使用示例
            * ![remoteSend调用示例](https://user-images.githubusercontent.com/50430941/133437744-863b70bf-955a-4594-883b-2010185711cf.png)
            * remoteSend的参数只有4个，与remoteCall的前4个参数一样
            * ![remoteSend请求返回](https://user-images.githubusercontent.com/50430941/133437780-3d0a5148-e3a5-4108-a91a-c3849d9d7a22.png)
    * ![协议包](https://user-images.githubusercontent.com/50430941/133437815-37d70d38-6ab0-498c-b6db-5ce7e2d0b7fa.png)
    * 组成2字节消息包长度+1字节整包标识+4字节session+data
        * 消息包长度：消息包的总字节数
        * 整包标识：0x80整包，0x81包头，0x82包体（可存在多个中间包），0x83包尾
        * sessoin：本次request的唯一key，由客户端生成
        * data：请求的数据
    * 数据大小与协议包
        * 消息包数据不大于32k
            * 整包标识值是0x80，data是要发送的数据
        * 消息包数据大于32k
            * 假设要发送的数据是96k，那么总共会发3个包
                * 整包标识值是0x81，data是0~32k的内容
                * 整包标识值是0x82，data是32~64k的内容
                * 整包标识值是0x83，data是64~96k的内容
* <b id="22">调用方式说明</b>
    * remoteCall需要返回，调用涉及的服务或者进程会保存调用缓存，当收到返回时，根据缓存处理返回结果。
    * remoteSend不返回，没有请求缓存，相比call，少了缓存数据和结果返回的一系列处理。
    * 补充：remoteSend的所有处理在remoteCall中都可以找到，所以用remoteCall来阐述跨进程的函数调用流程。
* <b id="11">发送方remoteCall调用流程</b>
    * ![request](https://user-images.githubusercontent.com/50430941/133438553-68e43312-9703-4a6f-98d1-1cdfd6cff7c5.png)
    * 客户端服务调用者（简称调用者）通过pushMessage给clusterclient发起远程调用，clusterclient检查请求参数中nodeName对应连接的status：
        * 如果status==disconnect，把请求放到requestCache中，走建立连接的流程，连接建立后，把请求发给远程服务器，把requestCache中的与这个服务端相关的请求生成唯一session后，然后把请求转移到session2context中
        * 如果status==connecting，把请求放到requestCache中，等连接完成的回调，回调中处理请求：发送请求，把requestCache中的与这个服务端相关的请求生成唯一session后，然后把请求转移到session2context中。
        * 如果status==connected，发送请求，把请求缓存到session2context
    * 收到网络返回消息包时，通过之前的请求缓存，原路返回给调用者
* <b id="12">接收方remoteCall响应流程</b>
    * ![request](https://user-images.githubusercontent.com/50430941/133437893-d5e19385-3490-4bbf-91a0-448a8484f4b9.png)
    * 当服务端的clusterMgr线程收到request的消息包时，会在参数列表上加上connectionId（与客户端连接实例的会话ID），把消息包push到clusterserver的消息队列中。
    * clusterserver收到request请求包，把参数解析成table后，根据参数中的servicename，把消息push到对于的服务的mq中。
    * 被调用者收到request请求后，根据参数method调用具体的函数，把函数返回结果经由被调用者->clusterserver->clusterMgr->网络传输，返回给客户端
## <b id="8">错误处理</b>
* 不管客户端还是服务端，套接字在加入event_base监听时，都会设置错误回调（Session::_error_cb_）和可读回调(Session::_read_cb_)函数，当某个端口发生网络错误时，回调Session::_error_cb_。
    * ![网络错误回调](https://user-images.githubusercontent.com/50430941/133437942-2bce3603-1211-4031-9619-669dc03b705b.png)
    * 客户端处理流程
        * 从tcpClient的_sessionDict_中移除报错的session实例，并销毁session实例。发消息通知clusterclient服务，clusterclient把对应的连接状态设置成disconnect，然后清除requestCache和session2context中报错节点的请求缓存，移除缓存的时候，需要一一发消息给调用者，给调用者返回错误码，让调用者移除缓存。
        * 示例，调用者处理远程调用返回结果
            ```
                function remoteCallback(session, code, rtbl)
                    local context = session2context[session]
                    if not context then
                        log.error("call fail, context not found.")
                        return
                    end
                    
                    session2context[session] = nil
                    if code == 0 then
                        -- dosomething
                    else
                        -- 错误处理
                    end
                end
            ```
    * ![网络错误处理_客户端](https://user-images.githubusercontent.com/50430941/133437988-cc1292ac-6ef4-4b12-995f-b467201d9b18.png)
    * 服务端处理流程
        * 从tcpServer的_sessionDict_中移除报错的session实例，并销毁session实例。如果之后clusterserver返回请求结果，在找不到对应的sessionId的情况下，丢弃这个请求。
## <b id="9">热更新</b>
* 后台gm指令发送到game进程
    * ![节点配置文件热更](https://user-images.githubusercontent.com/50430941/133438023-3c6877c6-1803-41da-9d29-07ba91d27fb6.png)
* 节点收到热更指令后，会做两件事
    * reWrite节点的配置文件clustername.lua，把最新的节点数据覆盖掉已有的clustername.lua文件内容。
    * reLoad节点的配置文件clustername.lua，并与原有的内存数据比较，如果新配置中没有原来的某个节点配置，则close对应的节点连接，并且触发onDisconnect事件。
## <b id="10">参考文档</b>
* [信长C++引擎文档](https://space.dingtalk.com/s/gwHOAu3kAQLOjJ-QYQPaACBkMGMyYWVlZDcxMTQ0M2M5OGFhY2YxNThlNzg4ZTdkYw) 密码: HyJr
