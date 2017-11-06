title: android-oem-lock-note
date: 2017-11-06 16:08:07
tags:
---

## 闲话

前阵子空闲时间学习了下 okhttp 源码，主要是重新学习 socket 网络编程，以及 okhttp 框架的一些特点。上周末，一个制作三星刷机包的朋友告诉我：国行三星手机通过官方 crom service 应用解锁手机后，才可以刷第三方刷机包，解锁后就保修就废了，很多人都不愿意解锁。于是朋友托我研究 crom service 上锁，如果研究成功，那么玩家更乐意解锁刷机。在研究上锁，刚好涉及到 android 底层 socket 通讯知识，同时分享下。

## socket 网络编程

在 okhttp 中，一次网络请求，在一条拦截链中完成，拦截链中每个拦截器完成比较少的任务（如重定向，失败重连，连接池，缓存等），很多大神也分享过 okhttp 拦截链的代码，这里就不重复介绍。而查阅 okhttp 一次 https 网络请求源码，我们可以发现一次请求中，可以简化分为以下四步：

|序号|方法|描述|
|---|---|---|
|1|connectSocket|通过 socket 连接远程服务器|
|2|connectTls|传输层安全协议，握手|
|3|writeRequest|写请求信息|
|4|readResponse|读响应信息|

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

上述代码，如果不关心**传输层安全协议**，上述代码看上去还是挺简单的，先连接远程服务器，然后建立 SSLSocket ，获取输入输出流，然后写请求信息，即可得到响应消息。满足 http 其中三个特点：1. C/S模式；2. 快速简单，支持多种请求方式； 3. 灵活，支持任意类型数据对象。


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

android 进程间通讯除了 binder 通讯，还有另外一种比较常见的 socket 通讯，在 android 的 native 层出现比较多。如果你想更深入 android 系统，应该克隆一份 aosp 源码。对着 android 系统源码，你将更容易理解其中奥妙。虽然本人 c++ 语言比较差，但看着代码也能理解其用意，和仿照系统源码学习编写 c++ 代码（jni编程）。

在 android 系统，socket 通讯为 c/s 模式，其中服务器在 init 进程 fork 各个 service 进程时创建，即 socket 服务器依附于 service 进程，而客户端在运行时连接服务器。

### 服务器创建

在 android 启动时，init 进程开始执行初始化，其中会解释 init.rc 并执行，在 init.rc 引入其他 rc 格式脚本文件，其中有服务声明，部分服务带有 socket 声明。以 vold.rc 为例：

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

**/system/core/init/init.cpp**

```c
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

## 参照

[HTTP请求头详解: http://blog.csdn.net/kfanning/article/details/6062118/](http://blog.csdn.net/kfanning/article/details/6062118/)
