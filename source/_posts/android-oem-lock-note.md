title: Android Socket 应用 - 网络编程与进程通讯
date: 2017-11-08 00:08:07
tags:
---

前阵子空闲时间学习了下 okhttp 源码，主要是重新学习 socket 网络编程，以及 okhttp 框架的一些特点。上周末，一个制作三星刷机包的朋友告诉我：国行三星手机通过官方 crom service 应用解锁手机后，才可以刷第三方刷机包，解锁后就保修就废了，很多人都不愿意解锁。于是朋友托我研究 crom service 上锁，如果研究成功，那么玩家更乐意解锁刷机。在研究上锁，刚好涉及到 android 底层 socket 通讯知识，同时分享下。

## socket 网络编程

在 okhttp 中，一次网络请求，在一条拦截链中完成，拦截链中每个拦截器完成比较少的任务（如重定向，失败重连，连接池，缓存等），很多大神也分享过拦截链的代码，就不重复介绍，这里主要介绍一次网络请求整个流程，可分以下四步：

|序号|方法|描述|
|---|---|---|
|1|connectSocket|通过 socket 连接远程服务器|
|2|connectTls|传输层安全协议，握手|
|3|writeRequest|写请求信息|
|4|readResponse|读响应信息|

<!-- more -->

以下代码为整合的一次 https 网络请求代码：

```java
public class Https implements Runnable {

    private static final int TIMEOUT = 5000;
    private static final int PORT = 443;

    private Socket socket;
    private Request request;
    private BufferedSink sink;
    private BufferedSource source;
    private Callback callback;

    public Https(Request request, Callback callback) {
        this.request = request;
        this.callback = callback;
    }

    @Override
    public void run() {
        try {
            connectSocket();
            connectTls();
            writeRequest();
            readResponse();
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            closeQuietly();
        }
    }

    private void connectSocket() throws IOException {
        socket = new Socket();
        socket.connect(new InetSocketAddress(request.url().host(), PORT), TIMEOUT);
    }

    private void connectTls() throws IOException {
        String host = request.url().host();
        X509TrustManager trustManager = systemDefaultTrustManager();
        SSLSocketFactory sslSocketFactory = systemDefaultSslSocketFactory(trustManager);
        SSLSocket sslSocket = (SSLSocket) sslSocketFactory.createSocket(socket, host, PORT, true);
        Platform.get().configureTlsExtensions(sslSocket, host, Util.immutableList(Protocol.HTTP_2, Protocol.HTTP_1_1));
        sslSocket.startHandshake();
        Handshake unverifiedHandshake = Handshake.get(sslSocket.getSession());
        if (!OkHostnameVerifier.INSTANCE.verify(host, sslSocket.getSession())) {
            sslSocket.close();
            return;
        }
        CertificatePinner.DEFAULT.check(host, unverifiedHandshake.peerCertificates());
        socket = sslSocket;
        sink = Okio.buffer(Okio.sink(socket.getOutputStream()));
        source = Okio.buffer(Okio.source(socket.getInputStream()));
    }

    private void rebuildRequest() throws IOException {
        Request.Builder requestBuilder = request.newBuilder();

        if (request.header("Host") == null) {
            requestBuilder.header("Host", Util.hostHeader(request.url(), false));
        }

        if (request.header("User-Agent") == null) {
            requestBuilder.header("User-Agent", Version.userAgent());
        }

        request = requestBuilder.build();
    }

    private void writeRequest() throws IOException {
        rebuildRequest();
        Headers headers = request.headers();
        sink.writeUtf8(RequestLine.get(request, Proxy.Type.HTTP)).writeUtf8("\r\n");
        for (int i = 0, size = headers.size(); i < size; i++) {
            sink.writeUtf8(headers.name(i)).writeUtf8(": ").writeUtf8(headers.value(i)).writeUtf8("\r\n");
        }
        sink.writeUtf8("\r\n");
        RequestBody requestBody = request.body();
        if (requestBody != null) {
            requestBody.writeTo(sink);
        }
        sink.close();
    }

    private void readResponse() throws IOException {
        StatusLine statusLine = StatusLine.parse(source.readUtf8LineStrict());
        Headers.Builder headers = new Headers.Builder();
        for (String line; (line = source.readUtf8LineStrict()).length() != 0; ) {
            int index = line.indexOf(":", 1);
            if (index != -1) {
                headers.add(line.substring(0, index), line.substring(index + 1));
            }
        }
        Response response = new Response.Builder()
                .request(request)
                .protocol(statusLine.protocol)
                .code(statusLine.code)
                .message(statusLine.message)
                .headers(headers.build())
                .build();
        if (callback != null) {
            byte[] data;
            if (!HttpHeaders.hasBody(response)) {
                data = new byte[0];
            } else {
                long contentLength = HttpHeaders.contentLength(response);
                if (contentLength != -1) {
                    long bytesRemaining = contentLength;
                    long bytesCount = 1024;
                    Buffer buffer = new Buffer();
                    while (bytesRemaining > 0) {
                        bytesCount = source.read(buffer, Math.min(bytesRemaining, bytesCount));
                        bytesRemaining -= bytesCount;
                    }
                    data = buffer.readByteArray();
                } else {
                    data = source.readByteArray();
                }
            }
            callback.response(data);
        }
    }

    private void closeQuietly() {
        try {
            if (socket != null) {
                socket.close();
            }
            if (sink != null) {
                sink.close();
            }
            if (source != null) {
                source.close();
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    private X509TrustManager systemDefaultTrustManager() {
        try {
            TrustManagerFactory trustManagerFactory = TrustManagerFactory.getInstance(TrustManagerFactory.getDefaultAlgorithm());
            trustManagerFactory.init((KeyStore) null);
            TrustManager[] trustManagers = trustManagerFactory.getTrustManagers();
            if (trustManagers.length != 1 || !(trustManagers[0] instanceof X509TrustManager)) {
                throw new IllegalStateException("Unexpected default trust managers:"
                        + Arrays.toString(trustManagers));
            }
            return (X509TrustManager) trustManagers[0];
        } catch (GeneralSecurityException e) {
            throw new AssertionError(); // The system has no TLS. Just give up.
        }
    }

    private SSLSocketFactory systemDefaultSslSocketFactory(X509TrustManager trustManager) {
        try {
            SSLContext sslContext = SSLContext.getInstance("TLS");
            sslContext.init(null, new TrustManager[]{trustManager}, null);
            return sslContext.getSocketFactory();
        } catch (GeneralSecurityException e) {
            throw new AssertionError(); // The system has no TLS. Just give up.
        }
    }

    public interface Callback {
        void response(byte[] data);
    }
}

```

上述代码，如果不关心**传输层安全协议**，看上去还是挺简单的，先连接远程服务器，然后建立 SSLSocket ，获取输入输出流，然后写请求信息，即可得到响应消息。满足 http 其中三个特点：1. C/S模式；2. 快速简单，支持多种请求方式； 3. 灵活，支持任意类型数据对象。


### 请求信息

请求格式：

```text
<request-line>
<headers>
<blank line>
[<request-body>]
```

1. 第一行，请求行，说明请求类型，请求资源路径以及 http 版本
2. 第二行开始为请求头信息，直至空行，一般包含** User-Agent **和** Host **
3. 空行
4. 最后为请求传输的内容，可能为空

举例：

```text
GET /repos/4ndroidev/DownloadManager/issues?state=closed HTTP/1.1
Host: api.github.com
User-Agent: okhttp/3.8.1

...
```

### 响应信息

响应格式：

```text
<status-line>
<headers>
<blank line>
[<response-body>]
```

举例：

```text
HTTP/1.1 200 OK
Server: GitHub.com
Date: Mon, 06 Nov 2017 11:23:53 GMT
Content-Type: application/json; charset=utf-8
Content-Length: 2546
Status: 200 OK
X-RateLimit-Limit: 60
X-RateLimit-Remaining: 46
X-RateLimit-Reset: 1509969526
Cache-Control: public, max-age=60, s-maxage=60
Vary: Accept
ETag: "7440f7e3c5e6da003d2ef12e02602ef2"
X-GitHub-Media-Type: github.v3; format=json
Access-Control-Expose-Headers: ETag, Link, Retry-After, X-GitHub-OTP, X-RateLimit-Limit, X-RateLimit-Remaining, X-RateLimit-Reset, X-OAuth-Scopes, X-Accepted-OAuth-Scopes, X-Poll-Interval
Access-Control-Allow-Origin: *
Content-Security-Policy: default-src 'none'
Strict-Transport-Security: max-age=31536000; includeSubdomains; preload
X-Content-Type-Options: nosniff
X-Frame-Options: deny
X-XSS-Protection: 1; mode=block
X-Runtime-rack: 0.066024
X-GitHub-Request-Id: 6821:536C:1C4665:50110F:5A004649

[{"url":"https://api.github.com/repos/4ndroidev/DownloadManager/issues/1","repository_url":"https://api.github.com/repos/4ndroidev/DownloadManager","labels_url":"https://api.github.com/repos/4ndroidev/DownloadManager/issues/1/labels{/name}","comments_url":"https://api.github.com/repos/4ndroidev/DownloadManager/issues/1/comments","events_url":"https://api.github.com/repos/4ndroidev/DownloadManager/issues/1/events","html_url":"https://github.com/4ndroidev/DownloadManager/issues/1","id":265718026,"number":1,"title":"State Change","user":{"login":"Thet-Tun-Aung","id":19220742,"avatar_url":"https://avatars1.githubusercontent.com/u/19220742?v=4","gravatar_id":"","url":"https://api.github.com/users/Thet-Tun-Aung","html_url":"https://github.com/Thet-Tun-Aung","followers_url":"https://api.github.com/users/Thet-Tun-Aung/followers","following_url":"https://api.github.com/users/Thet-Tun-Aung/following{/other_user}","gists_url":"https://api.github.com/users/Thet-Tun-Aung/gists{/gist_id}","starred_url":"https://api.github.com/users/Thet-Tun-Aung/starred{/owner}{/repo}","subscriptions_url":"https://api.github.com/users/Thet-Tun-Aung/subscriptions","organizations_url":"https://api.github.com/users/Thet-Tun-Aung/orgs","repos_url":"https://api.github.com/users/Thet-Tun-Aung/repos","events_url":"https://api.github.com/users/Thet-Tun-Aung/events{/privacy}","received_events_url":"https://api.github.com/users/Thet-Tun-Aung/received_events","type":"User","site_admin":false},"labels":[{"id":586505191,"url":"https://api.github.com/repos/4ndroidev/DownloadManager/labels/help%20wanted","name":"help wanted","color":"128A0C","default":true}],"state":"closed","locked":false,"assignee":null,"assignees":[],"milestone":null,"comments":2,"created_at":"2017-10-16T10:18:48Z","updated_at":"2017-10-17T08:56:54Z","closed_at":"2017-10-17T08:56:54Z","author_association":"NONE","body":"How to change state when file doesn't exit in storage? In order to re-download again.\r\n\r\nissue has been solved by following code:\r\n\r\nDownloadManager  manager = DownloadManager.getInstance();\r\n                        List<DownloadInfo> infos = manager.getAllInfo();\r\n                        for (DownloadInfo info : infos) {\r\n                            if (info.key.equals(task.key)) {\r\n                                manager.delete(info);\r\n                                break;\r\n                            }\r\n                        }\r\n\r\n                        downloadAdapter.notifyItemChanged(position);\r\n\r\n\r\nthanks @4ndroidev "}]
```

1. 第一行，状态行，说明 http 版本，响应状态码以及响应状态信息
2. 第二行开始为响应头信息，直至空行
3. 空行
4. 最后为响应内容，可能为空

---

## socket 进程通讯

android 进程间通讯除了 binder 通讯，还有另外一种比较常见的 socket 通讯，在 native 层出现比较多。如果你想更深入 android 系统，应该克隆一份 aosp 源码。对着系统源码，你将更容易理解其中奥妙。虽然本人 c++ 语言比较差，但看着代码也能理解其用意，和仿照系统源码学习编写 c++ 代码（jni编程）。文章接下来比较多贴代码的地方，需要仔细阅读**代码注释**。

在 android 系统，socket 通讯为 c/s 模式，其中服务器在 init 进程 fork 各个 service 进程时创建，即 socket 服务器依附于 service 进程，而客户端在运行时连接服务器。

一般情况下，我们只关注以下 socket 提供的函数，下文涉及。

```cpp
// 创建 socket
__socketcall int socket(int, int, int);

// 服务端关注的 bind，accept，listen 函数
__socketcall int bind(int, const struct sockaddr*, int);
__socketcall int accept(int, struct sockaddr*, socklen_t*);
__socketcall int listen(int, int);

// 客户端关注 connect 函数
__socketcall int connect(int, const struct sockaddr*, socklen_t);

// 服务端与客户端都关注的收发数据函数
extern ssize_t send(int, const void*, size_t, int);
extern ssize_t recv(int, void*, size_t, int);

// 当然还有一个关闭函数
close(int)

```

### 服务启动

这里描述的服务启动不单单是 socket 服务器启动，而是整个服务进程创建和启动过程，包含三个步骤：

1. 脚本解释
2. 启动服务
3. 程序运行

其中第二步，启动服务，才是真正从 init 进程 fork 出服务进程，并创建相关的 socket 服务器。

#### **脚本解释**

在 android 启动时，init 进程开始执行初始化，其中会解释 init.rc 并执行，在 init.rc 引入其他 rc 格式脚本文件，其中有服务声明，部分服务带有 socket 声明。研究三星上锁时涉及到 vold 进程，因此下文将围绕 vold 进程相关 socket 知识展开。

**/system/core/init/init.cpp**

```cpp
int main(int argc, char** argv) {
    ...
    // 获取解释器单例
    Parser& parser = Parser::GetInstance();
    // 创建 service, on, import 节点解释器
    parser.AddSectionParser("service",std::make_unique<ServiceParser>());
    parser.AddSectionParser("on", std::make_unique<ActionParser>());
    parser.AddSectionParser("import", std::make_unique<ImportParser>());
    // 解释执行 init.rc
    parser.ParseConfig("/init.rc");
    ...
}
```



**system/vold/vold.rc**

```sh
service vold /system/bin/vold \
        --blkid_context=u:r:blkid:s0 --blkid_untrusted_context=u:r:blkid_untrusted:s0 \
        --fsck_context=u:r:fsck:s0 --fsck_untrusted_context=u:r:fsck_untrusted:s0
    class core
    socket vold stream 0660 root mount
    socket cryptd stream 0660 root mount
    ioprio be 2
    writepid /dev/cpuset/foreground/tasks
```

脚本说明:

- 第1行，创建 vold 服务进程，其中二进制程序为 /system/bin/vold
- 第2-3行，启动 /system/bin/vold 程序的四个参数，这两行是接着第一行的，属于 service 节点参数
- 第4行，ServiceManager 会对相应类别的服务执行相应方法，略
- 第5行，创建名为 vold，类别为流的 socket 服务器，设置权限，用户及用户组，对应文件 /dev/socket/vold
- 第6行，创建名为 cryptd，类别为流的 socket 服务器，设置权限，用户及用户组，对应文件 /dev/socket/cryptd
- 第7行，设置 io 优先级，范围 0-7
- 第8行，fork 调用后得到的子进程号写入到文件 /dev/cpuset/foreground/tasks

服务解释器代码 - **system/core/init/service.cpp** :

```cpp
// 服务解释器可处理的参数列表
Service::OptionHandlerMap::Map& Service::OptionHandlerMap::map() const {
    constexpr std::size_t kMax = std::numeric_limits<std::size_t>::max();
    static const Map option_handlers = {
        {"class",       {1,     1,    &Service::HandleClass}},
        {"console",     {0,     0,    &Service::HandleConsole}},
        {"critical",    {0,     0,    &Service::HandleCritical}},
        {"disabled",    {0,     0,    &Service::HandleDisabled}},
        {"group",       {1,     NR_SVC_SUPP_GIDS + 1, &Service::HandleGroup}},
        {"ioprio",      {2,     2,    &Service::HandleIoprio}},
        {"keycodes",    {1,     kMax, &Service::HandleKeycodes}},
        {"oneshot",     {0,     0,    &Service::HandleOneshot}},
        {"onrestart",   {1,     kMax, &Service::HandleOnrestart}},
        {"seclabel",    {1,     1,    &Service::HandleSeclabel}},
        {"setenv",      {2,     2,    &Service::HandleSetenv}},
        {"socket",      {3,     6,    &Service::HandleSocket}},
        {"user",        {1,     1,    &Service::HandleUser}},
        {"writepid",    {1,     kMax, &Service::HandleWritepid}},
    };
    return option_handlers;
}

/** OptionHandlerMap 继承 KeywordMap，数据结构如下:
 * 
 * using FunctionInfo = std::tuple<std::size_t, std::size_t, Function>;
 * using Map = const std::map<std::string, FunctionInfo>;
 * 
 * {"class",{1,1,&Service::HandleClass}} 
 * 解释为遇到 class 节点，调用 HandleClass 函数，参数个数最少最多都为1
 *
 * 所有 HandleXXX 函数，基本上是收集信息参数，待 Service::Start 才正式使用这些参数信息
 */
...

// 遇到 service 节点，创建 service_ 对象，并解释参数
bool ServiceParser::ParseSection(const std::vector<std::string>& args,
                                 std::string* err) {
    if (args.size() < 3) {
        *err = "services must have a name and a program";
        return false;
    }

    const std::string& name = args[1];
    if (!IsValidName(name)) {
        *err = StringPrintf("invalid service name '%s'", name.c_str());
        return false;
    }

    std::vector<std::string> str_args(args.begin() + 2, args.end());
    service_ = std::make_unique<Service>(name, "default", str_args);
    return true;
}

// 逐行解释
bool ServiceParser::ParseLineSection(const std::vector<std::string>& args,
                                     const std::string& filename, int line,
                                     std::string* err) const {
    return service_ ? service_->HandleLine(args, err) : false;
}

// 添加服务到 ServiceManager
void ServiceParser::EndSection() {
    if (service_) {
        ServiceManager::GetInstance().AddService(std::move(service_));
    }
}

// 判断是否有效服务名称
bool ServiceParser::IsValidName(const std::string& name) const {
    if (name.size() > 16) {
        return false;
    }
    for (const auto& c : name) {
        if (!isalnum(c) && (c != '_') && (c != '-')) {
            return false;
        }
    }
    return true;
}
```

#### **启动服务**

服务启动代码 - **system/core/init/service.cpp** :

```cpp
bool Service::Start() {
    ...
    NOTICE("Starting service '%s'...\n", name_.c_str());

    // fork 写时拷贝，若成功调用一次则返回两个值，子进程返回0，父进程返回子进程标记
    pid_t pid = fork();

    // 子进程相关操作
    if (pid == 0) {
        umask(077);

        // 添加二进制程序运行时环境变量，扩展 ENV
        for (const auto& ei : envvars_) {
            add_environment(ei.name.c_str(), ei.value.c_str());
        }

        // 创建相关 socket 服务器
        for (const auto& si : sockets_) {
            int socket_type = ((si.type == "stream" ? SOCK_STREAM :
                                (si.type == "dgram" ? SOCK_DGRAM :
                                 SOCK_SEQPACKET)));
            const char* socketcon =
                !si.socketcon.empty() ? si.socketcon.c_str() : scon.c_str();

            int s = create_socket(si.name.c_str(), socket_type, si.perm,
                                  si.uid, si.gid, socketcon);
            if (s >= 0) {
                // 保存 socket 文件描述符到环境变量中
                PublishSocket(si.name, s);
            }
        }

        // 写进程号到相关文件
        std::string pid_str = StringPrintf("%d", getpid());
        for (const auto& file : writepid_files_) {
            if (!WriteStringToFile(pid_str, file)) {
                ERROR("couldn't write %s to %s: %s\n",
                      pid_str.c_str(), file.c_str(), strerror(errno));
            }
        }

        // 设置 io 优先级
        if (ioprio_class_ != IoSchedClass_NONE) {
            if (android_set_ioprio(getpid(), ioprio_class_, ioprio_pri_)) {
                ERROR("Failed to set pid %d ioprio = %d,%d: %s\n",
                      getpid(), ioprio_class_, ioprio_pri_, strerror(errno));
            }
        }

        if (needs_console) {
            setsid();
            OpenConsole();
        } else {
            ZapStdio();
        }

        setpgid(0, getpid());

        // 设置 gid
        if (gid_) {
            if (setgid(gid_) != 0) {
                ERROR("setgid failed: %s\n", strerror(errno));
                _exit(127);
            }
        }
        // 设置关联 gid
        if (!supp_gids_.empty()) {
            if (setgroups(supp_gids_.size(), &supp_gids_[0]) != 0) {
                ERROR("setgroups failed: %s\n", strerror(errno));
                _exit(127);
            }
        }
        // 设置uid
        if (uid_) {
            if (setuid(uid_) != 0) {
                ERROR("setuid failed: %s\n", strerror(errno));
                _exit(127);
            }
        }
        if (!seclabel_.empty()) {
            if (setexeccon(seclabel_.c_str()) < 0) {
                ERROR("cannot setexeccon('%s'): %s\n",
                      seclabel_.c_str(), strerror(errno));
                _exit(127);
            }
        }

        std::vector<std::string> expanded_args;
        std::vector<char*> strs;
        expanded_args.resize(args_.size());
        strs.push_back(const_cast<char*>(args_[0].c_str()));
        for (std::size_t i = 1; i < args_.size(); ++i) {
            if (!expand_props(args_[i], &expanded_args[i])) {
                ERROR("%s: cannot expand '%s'\n", args_[0].c_str(), args_[i].c_str());
                _exit(127);
            }
            strs.push_back(const_cast<char*>(expanded_args[i].c_str()));
        }
        strs.push_back(nullptr);

        // 在当前子进程启动二进制执行程序
        if (execve(strs[0], (char**) &strs[0], (char**) ENV) < 0) {
            ERROR("cannot execve('%s'): %s\n", strs[0], strerror(errno));
        }

        _exit(127);
    }

    if (pid < 0) {
        ERROR("failed to start '%s'\n", name_.c_str());
        pid_ = 0;
        return false;
    }
    ...
    return true;
}
```

创建 socket 服务器 - **system/core/init/util.cpp**
此处 socket 服务器仅完成了 bind 操作，而 listen 和 accept 操作在二进制程序运行时才执行

```cpp
int create_socket(const char *name, int type, mode_t perm, uid_t uid,
                  gid_t gid, const char *socketcon)
{
    struct sockaddr_un addr;
    int fd, ret, savederrno;
    char *filecon; // 自 Android 支持 SeLinux 之后很多代码都加入这些安全检测操作

    // 设置 selinux context
    if (socketcon) {
        if (setsockcreatecon(socketcon) == -1) {
            ERROR("setsockcreatecon(\"%s\") failed: %s\n", socketcon, strerror(errno));
            return -1;
        }
    }

    // 获取 socket 文件描述符
    fd = socket(PF_UNIX, type, 0);
    if (fd < 0) {
        ERROR("Failed to open socket '%s': %s\n", name, strerror(errno));
        return -1;
    }

    if (socketcon)
        setsockcreatecon(NULL);

    // 赋值 sockaddr_um，文件地址格式：/dev/socket/${name}
    memset(&addr, 0 , sizeof(addr));
    addr.sun_family = AF_UNIX;
    snprintf(addr.sun_path, sizeof(addr.sun_path), ANDROID_SOCKET_DIR"/%s",
             name);

    ret = unlink(addr.sun_path);
    if (ret != 0 && errno != ENOENT) {
        ERROR("Failed to unlink old socket '%s': %s\n", name, strerror(errno));
        goto out_close;
    }

    filecon = NULL;
    if (sehandle) {
        ret = selabel_lookup(sehandle, &filecon, addr.sun_path, S_IFSOCK);
        if (ret == 0)
            setfscreatecon(filecon);
    }

    // 绑定 socket 到目标地址
    ret = bind(fd, (struct sockaddr *) &addr, sizeof (addr));
    savederrno = errno;

    setfscreatecon(NULL);
    freecon(filecon);

    if (ret) {
        ERROR("Failed to bind socket '%s': %s\n", name, strerror(savederrno));
        goto out_unlink;
    }

    // 设置用户和用户组
    ret = lchown(addr.sun_path, uid, gid);
    if (ret) {
        ERROR("Failed to lchown socket '%s': %s\n", addr.sun_path, strerror(errno));
        goto out_unlink;
    }

    // 设置权限
    ret = fchmodat(AT_FDCWD, addr.sun_path, perm, AT_SYMLINK_NOFOLLOW);
    if (ret) {
        ERROR("Failed to fchmodat socket '%s': %s\n", addr.sun_path, strerror(errno));
        goto out_unlink;
    }

    INFO("Created socket '%s' with mode '%o', user '%d', group '%d'\n",
         addr.sun_path, perm, uid, gid);

    // 返回文件描述符，并存入启动二进制程序的环境变量 ENV 中
    return fd;

out_unlink:
    unlink(addr.sun_path);
out_close:
    close(fd);
    return -1;
}
```

#### **程序运行**

每个服务中 socket 服务器功能不一样，实现可以在 c++ 中，也可以在 java 中，因为 android 提供了 LocalServerSocket, LocalSocket 等类实现 socket 通讯，典型的 java 实现有应用进程的 ZygoteInit。这里以 vold 程序为例，在 c++ 中实现。

在 vold 程序运行时，会创建 VolumeManager 和 NetlinkManager， 并设置 CommandListener，而 CommandListener 是 socket 服务器关键，CommonListener 继承 FrameworkListener，FrameworkListener 继承 SocketListener。从名字可以推断，这里的 socket 通讯主要是用于服务端执行客户端发来的命令，并返回结果，接下来贴代码环节。

vold main 函数 - **system/vold/main.cpp**

```cpp
int main(int argc, char** argv) {
    ...
    VolumeManager *vm;
    CommandListener *cl;
    CryptCommandListener *ccl;
    NetlinkManager *nm;

    parse_args(argc, argv);

    sehandle = selinux_android_file_context_handle();
    if (sehandle) {
        selinux_android_set_sehandle(sehandle);
    }

    // Quickly throw a CLOEXEC on the socket we just inherited from init
    fcntl(android_get_control_socket("vold"), F_SETFD, FD_CLOEXEC);
    fcntl(android_get_control_socket("cryptd"), F_SETFD, FD_CLOEXEC);

    mkdir("/dev/block/vold", 0755);

    /* For when cryptfs checks and mounts an encrypted filesystem */
    klog_set_level(6);

    /* Create our singleton managers */
    if (!(vm = VolumeManager::Instance())) {
        LOG(ERROR) << "Unable to create VolumeManager";
        exit(1);
    }

    if (!(nm = NetlinkManager::Instance())) {
        LOG(ERROR) << "Unable to create NetlinkManager";
        exit(1);
    }

    if (property_get_bool("vold.debug", false)) {
        vm->setDebug(true);
    }

    // 初始化 CommandListener
    cl = new CommandListener();
    ccl = new CryptCommandListener();
    vm->setBroadcaster((SocketListener *) cl);
    nm->setBroadcaster((SocketListener *) cl);

    if (vm->start()) {
        PLOG(ERROR) << "Unable to start VolumeManager";
        exit(1);
    }

    if (process_config(vm)) {
        PLOG(ERROR) << "Error reading configuration... continuing anyways";
    }

    if (nm->start()) {
        PLOG(ERROR) << "Unable to start NetlinkManager";
        exit(1);
    }
    ...
}
```

SocketListener - **system/core/libsysutils/src/SocketListener.cpp**
职责：listen 和 创建线程进行 accept 客户端

```cpp
// 启动监听
int SocketListener::startListener(int backlog) {

    if (!mSocketName && mSock == -1) {
        SLOGE("Failed to start unbound listener");
        errno = EINVAL;
        return -1;
    } else if (mSocketName) {
        // 在 init 创建子进程时 create_socket 函数保存了 socket 的文件描述符到环境变量中，现在取出
        if ((mSock = android_get_control_socket(mSocketName)) < 0) {
            SLOGE("Obtaining file descriptor socket '%s' failed: %s",
                 mSocketName, strerror(errno));
            return -1;
        }
        SLOGV("got mSock = %d for %s", mSock, mSocketName);
        fcntl(mSock, F_SETFD, FD_CLOEXEC);
    }

    // socket 服务器进行监听
    if (mListen && listen(mSock, backlog) < 0) {
        SLOGE("Unable to listen on socket (%s)", strerror(errno));
        return -1;
    } else if (!mListen)
        mClients->push_back(new SocketClient(mSock, false, mUseCmdNum));

    if (pipe(mCtrlPipe)) {
        SLOGE("pipe failed (%s)", strerror(errno));
        return -1;
    }

    // 创建无限循环的线程，接受客户端的连入
    if (pthread_create(&mThread, NULL, SocketListener::threadStart, this)) {
        SLOGE("pthread_create (%s)", strerror(errno));
        return -1;
    }

    return 0;
}

// 结束监听
int SocketListener::stopListener() {
    ...
    if (mSocketName && mSock > -1) {
        // 关闭 socket 服务器
        close(mSock);
        mSock = -1;
    }
    ...
    return 0;
}

// 开启线程
void *SocketListener::threadStart(void *obj) {
    SocketListener *me = reinterpret_cast<SocketListener *>(obj);
    me->runListener();
    pthread_exit(NULL);
    return NULL;
}

// 无限循环接受客户端
void SocketListener::runListener() {

    SocketClientCollection pendingList;

    while(1) {
        ...
        if (mListen && FD_ISSET(mSock, &read_fds)) {
            sockaddr_storage ss;
            sockaddr* addrp = reinterpret_cast<sockaddr*>(&ss);
            socklen_t alen;
            int c;

            // 接受客户端连入
            do {
                alen = sizeof(ss);
                c = accept(mSock, addrp, &alen);
                SLOGV("%s got %d from accept", mSocketName, c);
            } while (c < 0 && errno == EINTR);
            if (c < 0) {
                SLOGE("accept failed (%s)", strerror(errno));
                sleep(1);
                continue;
            }
            fcntl(c, F_SETFD, FD_CLOEXEC);
            pthread_mutex_lock(&mClientsLock);
            mClients->push_back(new SocketClient(c, true, mUseCmdNum));
            pthread_mutex_unlock(&mClientsLock);
        }
        ...
    }
}

```

FrameworkListener - **system/core/libsysutils/src/FrameworkListener.cpp**
职责：主要负责收发数据，即接收和分发命令，并返回客户端执行结果

```cpp
FrameworkListener::FrameworkListener(const char *socketName, bool withSeq) :
                            SocketListener(socketName, true, withSeq) {
    init(socketName, withSeq);
}

// 接收命令数据
bool FrameworkListener::onDataAvailable(SocketClient *c) {
    char buffer[CMD_BUF_SIZE];
    int len;

    // 读取命令
    len = TEMP_FAILURE_RETRY(read(c->getSocket(), buffer, sizeof(buffer)));
    if (len < 0) {
        SLOGE("read() failed (%s)", strerror(errno));
        return false;
    } else if (!len) {
        return false;
    } else if (buffer[len-1] != '\0') {
        SLOGW("String is not zero-terminated");
        android_errorWriteLog(0x534e4554, "29831647");
        c->sendMsg(500, "Command too large for buffer", false);
        mSkipToNextNullByte = true;
        return false;
    }

    int offset = 0;
    int i;

    for (i = 0; i < len; i++) {
        if (buffer[i] == '\0') {
            if (mSkipToNextNullByte) {
                mSkipToNextNullByte = false;
            } else {
                // 分发命令和返回结果
                dispatchCommand(c, buffer + offset);
            }
            offset = i + 1;
        }
    }

    mSkipToNextNullByte = false;
    return true;
}

```

CommandListener - **system/vold/CommandListener.cpp**
职责：主要负责注册各种命令解释执行器，统一管理

```cpp
CommandListener::CommandListener() :
                 FrameworkListener("vold", true) {
    // 注册命令解释执行器
    registerCmd(new DumpCmd());
    registerCmd(new VolumeCmd());
    registerCmd(new AsecCmd());
    registerCmd(new ObbCmd());
    registerCmd(new StorageCmd());
    registerCmd(new FstrimCmd());
    registerCmd(new AppFuseCmd());
}
```

### 客户端连接

客户端可以在 c++ 或 java 中连接，在 vold.rc 中， socket vold 被声明了 0660 权限，即仅 root 用户， mount 组别的客户端才可以连接进行读写，若设有 selinux_label，要求更严格，关注系统安全的同学可以深入学习 SELinux 和 SEAndroid。

在源码中搜索了一番，发现只有 vdc.cpp 程序会发起 vold 的 socket 连接

```cpp
int main(int argc, char **argv) {
    int sock;
    ...
    const char* sockname = "vold";
    // 不一定 socket vold ，根据参数判断还可能是在同一服务的 socket crypted，可查看 vdc.rc
    if (!strcmp(argv[1], "cryptfs")) {
        sockname = "cryptd";
    }

    // 创建客户端，并连接
    while ((sock = socket_local_client(sockname,
                                 ANDROID_SOCKET_NAMESPACE_RESERVED,
                                 SOCK_STREAM)) < 0) {
        if (!wait_for_socket) {
            fprintf(stdout, "Error connecting to %s: %s\n", sockname, strerror(errno));
            exit(4);
        } else {
            usleep(10000);
        }
    }

    if (!strcmp(argv[1], "monitor")) {
        exit(do_monitor(sock, 0));
    } else {
        exit(do_cmd(sock, argc, argv));
    }
}

// 写命令
static int do_cmd(int sock, int argc, char **argv) {
    ...
    // 通过 sock 文件描述符写字节数据
    if (TEMP_FAILURE_RETRY(write(sock, cmd.c_str(), cmd.length() + 1)) < 0) {
        fprintf(stderr, "Failed to write command: %s\n", strerror(errno));
        return errno;
    }

    // 读取结果
    return do_monitor(sock, seq);
}

// 读取结果
static int do_monitor(int sock, int stop_after_seq) {
    char buffer[4096];
    int timeout = kCommandTimeoutMs;

    if (stop_after_seq == 0) {
        fprintf(stderr, "Connected to vold\n");
        timeout = -1;
    }

    while (1) {
        struct pollfd poll_sock = { sock, POLLIN, 0 };
        int rc = TEMP_FAILURE_RETRY(poll(&poll_sock, 1, timeout));
        ...
        memset(buffer, 0, sizeof(buffer));
        // 读取字节数据
        rc = TEMP_FAILURE_RETRY(read(sock, buffer, sizeof(buffer)));
        ...
    }
    return EIO;
}
```

## 引用

[HTTP请求头详解: http://blog.csdn.net/kfanning/article/details/6062118/](http://blog.csdn.net/kfanning/article/details/6062118/)

## 总结

本文主要介绍了 socket 网络编程和进程通讯相关概念和函数，具体的读写操作，协议并没有提及，事实上在 TCP/UDP 网络编程中，还需要考虑传输数据的顺序，边界及丢失等等。更多，请阅读[《Java+TCPIP+Socket编程(中文版).pdf》]()
