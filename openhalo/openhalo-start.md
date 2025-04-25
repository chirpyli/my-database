### openhalo启动过程分析
openhalo数据库是一款同时支持PostgreSQL协议和MySQL协议的开源数据库。本文分析其启动过程。让我们想一下，数据库怎么样才能同时支持两种协议呢？

#### 启动过程主流程
openhalo的启动过程与PostgreSQL启动过程基本一致，主要区别在于openhalo支持两种协议，所以需要创建两种协议处理函数，因支持同时存在MySQL客户端和PostgreSQL客户端的情况，所以需要数据库启动阶段即监听PostgreSQL端口，也监听MySQL端口。在创建监听socket时，需要为每个socket绑定对应的协议处理函数（mysql或者postgres），即设置socket的协议处理函数`ProtocolInterface`，这样当有客户端连接进来时，通过socket就可以判断出是PostgreSQL客户端还是MySQL客户端。

通过`select`函数监听socket，当有客户端连接过来时，
```c++
main(int argc, char *argv[])
--> MemoryContextInit();  // 创建TopMemoryContext
--> PostmasterMain(argc, argv);
    --> InitProcessGlobals(); // 初始化全局变量MyProcPid, MyStartTime， random seeds
    --> AllocSetContextCreate(TopMemoryContext, "Postmaster", ALLOCSET_DEFAULT_SIZES);  // 创建Postmaster内存上下文
    --> pqinitmask();  // 初始化信号屏蔽字
	--> PG_SETMASK(&BlockSig);  // 初始化时阻塞所有信号
        // 注册各种信号处理函数
    --> pqsignal_pm(SIGCHLD, reaper); 
    --> reset_shared();   // 创建共享内存和信号量
    --> for (i = 0; i < MAXLISTEN; i++)  // 初始化监听socket
		    ListenSocket[i] = PGINVALID_SOCKET;
        // 创建监听socket
    --> StreamServerPort(AF_UNSPEC, NULL, (unsigned short) PostPortNumber, NULL, ListenSocket, MAXLISTEN);
        --> fd = socket(addr->ai_family, SOCK_STREAM, 0)
        --> bind(fd, addr->ai_addr, addr->ai_addrlen);
        --> listen(fd, maxconn);
        --> setStandardProtocolSocket(fd);  // 将创建的socket放人ListenSocket数组中，这里创建的是标准PostgreSQL协议，
            --> setSocketProtocolHandler(index, &standard_protocol_handler); // 设置socket的协议处理函数，是PostgreSQL协议还是MySQL协议处理函数
    --> if (halo_mysql_listener_on)  // 如果配置了MySQL监听端口，则创建MySQL监听socket
            setSecondProtocolHandler(getSecondProtocolHandler());
            secondProtocolHandler->listen_init(); // 调用MySQL协议处理函数的初始化函数initListen(void)
            --> createServerPort(AF_UNSPEC, NULL, (unsigned short) PostMySQLPortNumber, NULL, &protocolHandler); // 创建MySQL监听socket
                --> fd = socket(addr->ai_family, SOCK_STREAM, 0)
                --> bind(fd, addr->ai_addr, addr->ai_addrlen);
                --> listen(fd, maxconn);
                --> setSecondProtocolSocket(fd); // 将创建的socker放入ListenSocket数组中，这里创建的是MySQL协议，
    --> ServerLoop();
        --> initMasks(&readmask); // 初始化监听socket集合
        --> for (;;)
            {
                int selres = select(nSockets, &rmask, NULL, NULL, &timeout); // 多路复用监听socket，阻塞在这里，直到有socket连接到达

                if (selres > 0)  // 有socket连接到达
                {
                    for (i = 0; i < MAXLISTEN; i++)
			        {
                        if (FD_ISSET(ListenSocket[i], &rmask))
                        {
                            Port *port = ConnCreate(ListenSocket[i]);
                            if (port)
                            { 
                                BackendStartup(port);

                                /* We no longer need the open socket or port structure in this process*/
                                StreamClose(port->sock);
                                ConnFree(port);
                            }
                        }
                    }
                }
            }
```

##### 如何判断是postgres客户端连接还是mysql客户端连接？
通过socket的协议处理函数`ProtocolInterface`判断，如果协议处理函数是`standard_protocol_handler`，则认为是PostgreSQL客户端连接，否则认为是MySQL客户端连接。具体的，与`ListenSocket`数组一一对应的协议处理函数数组`ListenHandler`，每向`ListenSocket`数组中添加一个socket时，都会向`ListenHandler`数组中添加一个协议处理函数。
```c++
static const ProtocolInterface    *ListenHandler[MAXLISTEN];
```
具体的可以分析`setStandardProtocolSocket`函数以及`setSecondProtocolSocket`函数。


##### mysql连接处理流程
当用户通过`mysql -P 3306 -h 127.0.0.1`连接数据库时，数据库服务端的`select`函数会返回，开始创建连接流程。这里需要注意是通过MySQL协议来与数据库交互的，需要解析MySQL协议，以及通过MySQL的认证方式。建立连接后，解析客户端发来的数据包，根据MySQL协议进行解析，然后根据解析结果进行相应的处理。

```c++
ServerLoop(void)
--> select(nSockets, &rmask, NULL, NULL, &timeout);  // 监听socket
--> ConnCreate(ListenSocket[i]);
    --> Port *port = (Port *) calloc(1, sizeof(Port)) // 创建Port结构体
    --> port->protocol_handler = getProtocolHandlerByFd(serverFd); // 获取socket对应的协议处理函数, 这里的protocol_handler为openhalo新增的字段，postgres中没有该字段
    --> port->protocol_handler->accept(serverFd, port) // 调用acceptConn函数
        --> StreamConnection(serverFd, port);
            --> port->sock = accept(server_fd, (struct sockaddr *) &port->raddr.addr, &port->raddr.salen)
--> BackendStartup(port);
    --> pid_t pid = fork_process();  // 创建子进程，处理连接
    --> if (pid == 0)  // 子进程处理连接    
        {
            BackendInitialize(port);
            --> port->protocol_handler->init(); // 调用initServer函数
                --> initNetTransceiver();
                --> FeBeWaitSet = CreateWaitEventSet(TopMemoryContext, 3); // 创建等待事件集合
                --> AddWaitEventToSet(FeBeWaitSet, WL_SOCKET_WRITEABLE, MyProcPort->sock, NULL, NULL);  // 添加等待事件,套接字可写事件
                --> AddWaitEventToSet(FeBeWaitSet, WL_LATCH_SET, PGINVALID_SOCKET, MyLatch, NULL);
                --> AddWaitEventToSet(FeBeWaitSet, WL_POSTMASTER_DEATH, PGINVALID_SOCKET, NULL, NULL);
            --> port->protocol_handler->start(port);  // 调用startServer函数
            BackendRun(port);
            --> port->protocol_handler->mainfunc(port, ac, av); // 调用 PostgresMain函数
                --> BaseInit();
                --> InitProcess();
                --> InitPostgres(dbname, InvalidOid, username, InvalidOid, NULL, false);
                    --> MyProcPort->protocol_handler->authenticate(MyProcPort, &username);  // 客户端认证
                    --> InitParserEngine();   // 解析引擎
                    --> InitPlannerEngine();  // 优化器引擎
                    --> InitExecutorEngine(); // 执行引擎
                --> MyProcPort->protocol_handler->send_ready_for_query(whereToSendOutput)
                --> ReadCommand(&input_message);
                    --> MyProcPort->protocol_handler->read_command(inBuf);
                        --> netTransceiver->readPayload(inBuf)
                            --> readAndProcessPacket(inBuf) // 处理MySQL协议包
                                --> readAndProcessPacketHeader();
                                    --> readBytes(buff, EACH_PACKET_HEARDER_LENGTH)
                                        --> secure_read(netTransceiver->mysPort, buff, len);
                                            --> secure_raw_read(port, ptr, len);
                                                --> recv(port->sock, ptr, len, 0);
                                --> readPacketPayload(inBuf, payloadLen);
        }

```