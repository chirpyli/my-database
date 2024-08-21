## PostgreSQL源码分析——auth_delay

auth_delay是PostgreSQL中用于防止暴力破解密码的插件，在配置文件postgresql.conf中，auth_delay参数可以设置延迟时间，单位为毫秒。

其防止暴力破解的原理是，当认证失败时，会延时一段时间，然后返回认证失败。这样，攻击者就无法在短时间内尝试大量的密码，从而降低了暴力破解的成功率。

### auth_delay配置
配置认证失败延时时间：
```shell
shared_preload_libraries = 'auth_delay'
auth_delay.milliseconds = 1000 # 设置认证失败时延1秒
```
测试：
```sql
postgres@slpc:~/works/my-github/my-database$ psql -h 192.168.109.133 -U hangzhou
Password for user hangzhou:  -- 输入错误密码后，经过1秒才返回错误信息    
psql: error: connection to server at "192.168.109.133", port 5432 failed: FATAL:  password authentication failed for user "hangzhou"
```

### 实现原理
关于口令认证的过程，可参考文章[PostgreSQL源码分析——口令认证](https://www.modb.pro/db/1752960246943404032)

每当用户连接数据库时，postmaster进程会先创建一个postgres子进程，在postgres子进程中`InitPostgres`阶段会调用`ClientAuthentication`函数进行认证。在`ClientAuthentication`函数的最后，有一个钩子函数`ClientAuthentication_hook`，用于调用插件中的认证函数。auth_delay插件就是通过这个钩子函数实现的。

```c++
extern void ClientAuthentication(Port *port);

/* Hook for plugins to get control in ClientAuthentication() */
typedef void (*ClientAuthentication_hook_type) (Port *, int);
extern PGDLLIMPORT ClientAuthentication_hook_type ClientAuthentication_hook;

```
调用栈如下：
```c++
auth_delay.so!auth_delay_checks(Port * port, int status) (contrib\auth_delay\auth_delay.c:48)
ClientAuthentication(Port * port) (src\backend\libpq\auth.c:659)
PerformAuthentication(Port * port) (src\backend\utils\init\postinit.c:241)
InitPostgres(const char * in_dbname, Oid dboid, const char * username, Oid useroid, char * out_dbname, _Bool override_allow_connections) (src\backend\utils\init\postinit.c:782)
PostgresMain(int argc, char ** argv, const char * dbname, const char * username) (src\backend\tcop\postgres.c:4168)
BackendRun(Port * port) (src\backend\postmaster\postmaster.c:4540)
BackendStartup(Port * port) (src\backend\postmaster\postmaster.c:4262)
ServerLoop() (src\backend\postmaster\postmaster.c:1748)
PostmasterMain(int argc, char ** argv) (src\backend\postmaster\postmaster.c:1420)
main(int argc, char ** argv) (src\backend\main\main.c:209)
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

    // 钩子函数，在这里进行检查，如果认证失败，则延时
	if (ClientAuthentication_hook)
		(*ClientAuthentication_hook) (port, status);

	if (status == STATUS_OK)
		sendAuthRequest(port, AUTH_REQ_OK, NULL, 0);
	else
		auth_failed(port, status, logdetail);
}
```

### auth_delay源码
```c++
void		_PG_init(void);

/* GUC Variables */
static int	auth_delay_milliseconds;

/* Original Hook */
static ClientAuthentication_hook_type original_client_auth_hook = NULL;

/*
 * Check authentication
 */
static void
auth_delay_checks(Port *port, int status)
{
	/*
	 * Any other plugins which use ClientAuthentication_hook.
	 */
	if (original_client_auth_hook)
		original_client_auth_hook(port, status);

	/*
	 * Inject a short delay if authentication failed.
	 */
    // 认证失败的时候，进行延时，防止暴力破解
	if (status != STATUS_OK)
	{
		pg_usleep(1000L * auth_delay_milliseconds);
	}
}

/*
 * Module Load Callback
 */
void
_PG_init(void)
{
	/* Define custom GUC variables */
	DefineCustomIntVariable("auth_delay.milliseconds",
							"Milliseconds to delay before reporting authentication failure",
							NULL,
							&auth_delay_milliseconds,
							0,
							0, INT_MAX / 1000,
							PGC_SIGHUP,
							GUC_UNIT_MS,
							NULL,
							NULL,
							NULL);
	/* Install Hooks */
	original_client_auth_hook = ClientAuthentication_hook;
	ClientAuthentication_hook = auth_delay_checks;
}
```