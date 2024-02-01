### PostgreSQL源码分析——口令认证

#### 认证机制
对于数据库系统来说，其作为服务端，接受来自客户端的请求。对此，必须有对客户端的认证机制，只有通过身份认证的客户端才可以访问数据库资源，防止非法用户连接数据库。PostgreSQL支持认证方法有很多：
```c++
typedef enum UserAuth
{
	uaReject,
	uaImplicitReject,			/* Not a user-visible option */
	uaTrust,
	uaIdent,
	uaPassword,
	uaMD5,
	uaSCRAM,
	uaGSS,
	uaSSPI,
	uaPAM,
	uaBSD,
	uaLDAP,
	uaCert,
	uaRADIUS,
	uaPeer
#define USER_AUTH_LAST uaPeer	/* Must be last value of this enum */
} UserAuth;
```
对此，我们仅分析常用的口令认证方式。目前PostgreSQL支持MD5、SCRAM-SHA-256对密码哈希和身份验证支持，建议使用SCRAM-SHA-256，相较MD5安全性更高。不同的基于口令的认证方法的可用性取决于用户的口令在服务器上是如何被加密（或者更准确地说是哈希）的。这由设置口令时的配置参数password_encryption控制。可用值为`scham-sha-256`、`md5`。
```shell
host    all             all             0.0.0.0/0               scham-sha-256
```


#### 源码分析
这里只分析口令认证的情况。 口令认证分为明文口令认证和加密口令认证。明文口令认证要求客户端提供一个未加密的口令进行认证，安全性较差，已经被禁止使用。加密口令认证要求客户端提供一个经过SCRAM-SHA-256加密的口令进行认证，该口令在传送过程中使用了结合salt的单向哈希加密，增强了安全性。口令都存储在pg_authid系统表中，可通过`CREATE USER test WITH PASSWORD '******';`等命令进行设置。

##### 客户端
以psql客户端为例，当客户端发出连接请求后，分析口令认证的过程。
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

##### 服务端
服务端相关流程：
```c++
main()
--> PostmasterMain(argc, argv);
    --> InitProcessGlobals();   // set MyProcPid, MyStartTime[stamp], random seeds
        --> pg_prng_strong_seed(&pg_global_prng_state)
        --> srandom(pg_prng_uint32(&pg_global_prng_state));
    --> InitializeGUCOptions();
    --> ProcessConfigFile(PGC_POSTMASTER);
        --> ProcessConfigFileInternal(context, true, elevel);
            --> ParseConfigFp(fp, abs_path, depth, elevel, head_p, tail_p);
    --> load_hba()  // 读pg_hba.conf
        --> parse_hba_line(tok_line, LOG)   // 解析pg_hba.conf中的每条记录,解析到HbaLine的链表结构中
    --> load_ident()
    --> ServerLoop();
        --> BackendStartup(port);
            --> InitPostmasterChild();
            --> BackendInitialize(port);
                --> pq_init();
                --> RegisterTimeout(STARTUP_PACKET_TIMEOUT, StartupPacketTimeoutHandler);
	            --> enable_timeout_after(STARTUP_PACKET_TIMEOUT, AuthenticationTimeout * 1000);
                --> ProcessStartupPacket(port, false, false);   // Read a client's startup packet
                    --> pq_startmsgread();
                    --> pq_getbytes((char *) &len, 1)
            --> InitProcess();
            --> BackendRun(port);
                --> PostgresMain(port->database_name, port->user_name);
                    --> BaseInit();
                    // 在接收用户SQL请求前进行认证
                    --> InitPostgres(dbname, InvalidOid, username, InvalidOid,!am_walsender, false, NULL);
                        --> PerformAuthentication(MyProcPort);
                            --> enable_timeout_after(STATEMENT_TIMEOUT, AuthenticationTimeout * 1000);
                            --> ClientAuthentication(port);
                                --> hba_getauthmethod(port);
                                    --> check_hba(port);
                                    // 根据不同的认证方法进行认证
                                --> switch (port->hba->auth_method)
	                                {		
                                        case uaTrust:
			                                    status = STATUS_OK;
			                                    break;
                                        case uaMD5:
		                                case uaSCRAM:
                                                // 口令认证
			                                    status = CheckPWChallengeAuth(port, &logdetail);
                                                         --> 
			                                    break;
                                        // ...
                                    }
                                --> if (status == STATUS_OK)
		                                sendAuthRequest(port, AUTH_REQ_OK, NULL, 0);
	                                else
		                                auth_failed(port, status, logdetail);
                            --> disable_timeout(STATEMENT_TIMEOUT, false);
                        --> initialize_acl();
                        // 认证通过后才能接受用户的SQL请求
                    --> for (;;)
                        {
                            // ...
                            exec_simple_query(query_string);
                        }
```

每次客户端发起连接时，postgres主进程都会fork一个子进程：
```c++
static int BackendStartup(Port *port)
{
    // ...
	pid = fork_process();
	if (pid == 0)				/* child */
	{
		free(bn);

		/* Detangle from postmaster */
		InitPostmasterChild();

		/* Close the postmaster's sockets */
		ClosePostmasterPorts(false);

		/* Perform additional initialization and collect startup packet */
		BackendInitialize(port);

		/*
		 * Create a per-backend PGPROC struct in shared memory. We must do
		 * this before we can use LWLocks. In the !EXEC_BACKEND case (here)
		 * this could be delayed a bit further, but EXEC_BACKEND needs to do
		 * stuff with LWLocks before PostgresMain(), so we do it here as well
		 * for symmetry.
		 */
		InitProcess();  

		/* And run the backend */
		BackendRun(port);  // 
	}
}
```
每个子进程在接收用户的SQL请求前，需要先进行认证。主要实现在`ClientAuthentication`函数中，通过认证才能继续进行下一步。
```c++
/*
 * Client authentication starts here.  If there is an error, this
 * function does not return and the backend process is terminated. */
void ClientAuthentication(Port *port)
{
	int			status = STATUS_ERROR;
	const char *logdetail = NULL;

	hba_getauthmethod(port);


	/*
	 * This is the first point where we have access to the hba record for the
	 * current connection, so perform any verifications based on the hba
	 * options field that should be done *before* the authentication here. */
	if (port->hba->clientcert != clientCertOff)
	{
		/* If we haven't loaded a root certificate store, fail */
		if (!secure_loaded_verify_locations())
			ereport(FATAL,
					(errcode(ERRCODE_CONFIG_FILE_ERROR),
					 errmsg("client certificates can only be checked if a root certificate store is available")));

		/* If we loaded a root certificate store, and if a certificate is
		 * present on the client, then it has been verified against our root
		 * certificate store, and the connection would have been aborted
		 * already if it didn't verify ok. */
		if (!port->peer_cert_valid)
			ereport(FATAL,(errcode(ERRCODE_INVALID_AUTHORIZATION_SPECIFICATION), errmsg("connection requires a valid client certificate")));
	}

    // 根据不同的认证方法，进行不同的处理
	switch (port->hba->auth_method)
	{
        // ...
		case uaMD5:
		case uaSCRAM:
			status = CheckPWChallengeAuth(port, &logdetail);
			break;

		case uaPassword:
			status = CheckPasswordAuth(port, &logdetail);
			break;

		case uaTrust:
			status = STATUS_OK;
			break;
	}

	if ((status == STATUS_OK && port->hba->clientcert == clientCertFull)|| port->hba->auth_method == uaCert)
	{
		/*
		 * Make sure we only check the certificate if we use the cert method
		 * or verify-full option.
		 */
#ifdef USE_SSL
		status = CheckCertAuth(port);
#else
		Assert(false);
#endif
	}

	if (ClientAuthentication_hook)
		(*ClientAuthentication_hook) (port, status);

	if (status == STATUS_OK)
		sendAuthRequest(port, AUTH_REQ_OK, NULL, 0);
	else
		auth_failed(port, status, logdetail);
}
```
对于口令认证，具体实现为`CheckPWChallengeAuth`函数。
```c++
/* MD5 and SCRAM authentication. */
static int CheckPWChallengeAuth(Port *port, const char **logdetail)
{
	int			auth_result;
	char	   *shadow_pass;
	PasswordType pwtype;

	/* First look up the user's password. */ // 查找pg_authid系统表中的用户密码
	shadow_pass = get_role_password(port->user_name, logdetail);

	if (!shadow_pass)
		pwtype = Password_encryption;
	else
		pwtype = get_password_type(shadow_pass);

	/* If 'md5' authentication is allowed, decide whether to perform 'md5' or
	 * 'scram-sha-256' authentication based on the type of password the user
	 * has.  If it's an MD5 hash, we must do MD5 authentication, and if it's a
	 * SCRAM secret, we must do SCRAM authentication.
	 *
	 * If MD5 authentication is not allowed, always use SCRAM.  If the user
	 * had an MD5 password, CheckSASLAuth() with the SCRAM mechanism will fail. */
	if (port->hba->auth_method == uaMD5 && pwtype == PASSWORD_TYPE_MD5)
		auth_result = CheckMD5Auth(port, shadow_pass, logdetail);
	else
		auth_result = CheckSASLAuth(&pg_be_scram_mech, port, shadow_pass,logdetail);

	if (shadow_pass)
		pfree(shadow_pass);

	/* If get_role_password() returned error, return error, even if the authentication succeeded. */
	if (!shadow_pass)
	{
		Assert(auth_result != STATUS_OK);
		return STATUS_ERROR;
	}

	if (auth_result == STATUS_OK)
		set_authn_id(port, port->user_name);

	return auth_result;
}

int CheckSASLAuth(const pg_be_sasl_mech *mech, Port *port, char *shadow_pass, const char **logdetail)
{
	StringInfoData sasl_mechs;
	int			mtype;
	StringInfoData buf;
	void	   *opaq = NULL;
	char	   *output = NULL;
	int			outputlen = 0;
	const char *input;
	int			inputlen;
	int			result;
	bool		initial;

	/*
	 * Send the SASL authentication request to user.  It includes the list of
	 * authentication mechanisms that are supported.
	 */
	initStringInfo(&sasl_mechs);

	mech->get_mechanisms(port, &sasl_mechs);
	/* Put another '\0' to mark that list is finished. */
	appendStringInfoChar(&sasl_mechs, '\0');

	sendAuthRequest(port, AUTH_REQ_SASL, sasl_mechs.data, sasl_mechs.len);
	pfree(sasl_mechs.data);

	/*
	 * Loop through SASL message exchange.  This exchange can consist of
	 * multiple messages sent in both directions.  First message is always
	 * from the client.  All messages from client to server are password
	 * packets (type 'p').
	 */
	initial = true;
	do
	{
		pq_startmsgread();
		mtype = pq_getbyte();
		if (mtype != 'p')
		{
			/* Only log error if client didn't disconnect. */
			if (mtype != EOF)
			{
				ereport(ERROR,(errcode(ERRCODE_PROTOCOL_VIOLATION), errmsg("expected SASL response, got message type %d",mtype)));
			}
			else
				return STATUS_EOF;
		}

		/* Get the actual SASL message */
		initStringInfo(&buf);
		if (pq_getmessage(&buf, PG_MAX_SASL_MESSAGE_LENGTH))
		{
			/* EOF - pq_getmessage already logged error */
			pfree(buf.data);
			return STATUS_ERROR;
		}

		elog(DEBUG4, "processing received SASL response of length %d", buf.len);

		/* The first SASLInitialResponse message is different from the others.
		 * It indicates which SASL mechanism the client selected, and contains
		 * an optional Initial Client Response payload.  The subsequent
		 * SASLResponse messages contain just the SASL payload. */
		if (initial)
		{
			const char *selected_mech;
			selected_mech = pq_getmsgrawstring(&buf);

			/*
			 * Initialize the status tracker for message exchanges.
			 *
			 * If the user doesn't exist, or doesn't have a valid password, or
			 * it's expired, we still go through the motions of SASL
			 * authentication, but tell the authentication method that the
			 * authentication is "doomed". That is, it's going to fail, no matter what.
			 *
			 * This is because we don't want to reveal to an attacker what
			 * usernames are valid, nor which users have a valid password. */
			opaq = mech->init(port, selected_mech, shadow_pass);

			inputlen = pq_getmsgint(&buf, 4);
			if (inputlen == -1)
				input = NULL;
			else
				input = pq_getmsgbytes(&buf, inputlen);

			initial = false;
		}
		else
		{
			inputlen = buf.len;
			input = pq_getmsgbytes(&buf, buf.len);
		}
		pq_getmsgend(&buf);

		/* Hand the incoming message to the mechanism implementation. */
		result = mech->exchange(opaq, input, inputlen,&output, &outputlen,logdetail);

		/* input buffer no longer used */
		pfree(buf.data);

		if (output)
		{
			/*PG_SASL_EXCHANGE_FAILURE with some output is forbidden by SASL.
			  Make sure here that the mechanism used got that right. */
			if (result == PG_SASL_EXCHANGE_FAILURE)
				elog(ERROR, "output message found after SASL exchange failure");

			/* Negotiation generated data to be sent to the client. */
			elog(DEBUG4, "sending SASL challenge of length %d", outputlen);

			if (result == PG_SASL_EXCHANGE_SUCCESS)
				sendAuthRequest(port, AUTH_REQ_SASL_FIN, output, outputlen);
			else
				sendAuthRequest(port, AUTH_REQ_SASL_CONT, output, outputlen);

			pfree(output);
		}
	} while (result == PG_SASL_EXCHANGE_CONTINUE);

	/* Oops, Something bad happened */
	if (result != PG_SASL_EXCHANGE_SUCCESS)
	{
		return STATUS_ERROR;
	}

	return STATUS_OK;
}

```

