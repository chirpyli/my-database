### PostgreSQL源码分析——psql
psql是一个PostgreSQL数据库自带的客户端工具，用来与数据库进行交互，当然，你也可以用其他工具，一般是作为开发测试调试用，或者管理员使用，实际生产环境一般通过libpq库或者JDBC等与数据库进行交互。psql本质上就是一个libpq客户端，通过libpq与PostgreSQL服务端进行交互。


#### 主流程如下
psql的核心功能，连接数据库，执行用户的命令，其中连接到数据库需要经过认证，这里只分析口令认证的情况。加密口令认证要求客户端提供一个经过MD5加密的口令进行认证，该口令在传送过程中使用了结合salt的单向MD5加密，增强了安全性。具体的可参考官方文档[数据库连接控制函数](http://www.postgres.cn/docs/14/libpq-connect.html)。

```c++
main
--> parse_psql_options(argc, argv, &options);
    // 连接数据库，中间会有认证这块，有不同的认证方法，这里列的是口令认证方式的流程
--> do 
    {
         // 输入host、port、user、password、dbname等信息
         PQconnectdbParams(keywords, values, true);
         --> PQconnectStartParams(keywords, values, expand_dbname);
             --> makeEmptyPGconn();
             --> conninfo_array_parse(keywords, values, &conn->errorMessage, true, expand_dbname);
             --> fillPGconn(conn, connOptions)
             --> connectDBStart(conn)
                 --> PQconnectPoll(conn)    // 尝试建立TCP连接
                     --> socket(addr_cur->ai_family, SOCK_STREAM, 0);
                     --> connect(conn->sock, addr_cur->ai_addr, addr_cur->ai_addrlen)
         --> connectDBComplete(conn);
             // 认证这块与数据库服务端交互以及状态很多，这里没有全部列出。
             --> PQconnectPoll(conn);
                 --> pqReadData(conn);
                     --> pqsecure_read(conn, conn->inBuffer + conn->inEnd, conn->inBufSize - conn->inEnd);
                         --> pqsecure_raw_read(conn, ptr, len);
                             --> recv(conn->sock, ptr, len, 0);
                 --> pg_fe_sendauth(areq, msgLength, conn);
                     --> pg_SASL_init(conn, payloadlen)
                         --> conn->sasl->init(conn,password,selected_mechanism);
                             --> pg_saslprep(password, &prep_password);
                         --> conn->sasl->exchange(conn->sasl_state, NULL, -1, &initialresponse, &initialresponselen, &done, &success);
                             --> build_client_first_message(state);
                                 --> pg_strong_random(raw_nonce, SCRAM_RAW_NONCE_LEN)
         PQconnectionNeedsPassword(pset.db)     // 是否需要密码，如果需要，等待用户数据密码

    }
    // 进入主循环，等待用户输入命令，执行命令
--> MainLoop(stdin);  
    --> psql_scan_create(&psqlscan_callbacks);
    --> while (successResult == EXIT_SUCCESS)
        {
            // 等待用户输入命令
            gets_interactive(get_prompt(prompt_status, cond_stack), query_buf);

            SendQuery(query_buf->data); // 把用户输入的SQL命令发送到数据库服务
            --> ExecQueryAndProcessResults(query, &elapsed_msec, &svpt_gone, false, NULL, NULL) > 0);
                --> PQsendQuery(pset.db, query);    // 通过libpq与数据库交互
                --> PQgetResult(pset.db);
        }
```
