[![Build Status](https://travis-ci.org/ithewei/libhv.svg?branch=master)](https://travis-ci.org/ithewei/libhv)

## Intro

Like `libevent, libev, and libuv`,
`libhv` provides event-loop with non-blocking IO and timer,
but simpler apis and richer protocols.

## Features

- cross-platform (Linux, Windows, Mac)
- event-loop (IO, timer, idle)
- http client/server (include https http1/x http2 grpc)
- protocols
    - dns
    - ftp
    - smtp
- apps
    - ls
    - ifconfig
    - ping
    - nc
    - nmap
    - nslookup
    - ftp
    - sendmail
    - httpd
    - curl

## Getting Started

### http
see examples/httpd.cpp
```
#include "HttpServer.h"

int http_api_hello(HttpRequest* req, HttpResponse* res) {
    res->body = "hello";
    return 0;
}

int main() {
    HttpService service;
    service.base_url = "/v1/api";
    service.AddApi("/hello", HTTP_GET, http_api_hello);

    http_server_t server;
    server.port = 8080;
    server.worker_processes = 4;
    server.service = &service;
    http_server_run(&server);
    return 0;
}
```

```shell
git clone https://github.com/ithewei/libhv.git
cd libhv
make httpd curl

bin/httpd -d
ps aux | grep httpd

# http web service
bin/curl -v localhost:8080

# indexof
bin/curl -v localhost:8080/downloads/

# http api service
bin/curl -v -X POST localhost:8080/v1/api/json -H "Content-Type:application/json" -d '{"user":"admin","pswd":"123456"}'

# webbench (linux only)
make webbench
bin/webbench -c 2 -t 60 localhost:8080
```

### event-loop
see examples/tcp.c
```
#include "hloop.h"
#include "hsocket.h"

#define RECV_BUFSIZE    8192
static char recvbuf[RECV_BUFSIZE];

void on_close(hio_t* io) {
    printf("on_close fd=%d error=%d\n", hio_fd(io), hio_error(io));
}

void on_recv(hio_t* io, void* buf, int readbytes) {
    printf("on_recv fd=%d readbytes=%d\n", hio_fd(io), readbytes);
    char localaddrstr[SOCKADDR_STRLEN] = {0};
    char peeraddrstr[SOCKADDR_STRLEN] = {0};
    printf("[%s] <=> [%s]\n",
            SOCKADDR_STR(hio_localaddr(io), localaddrstr),
            SOCKADDR_STR(hio_peeraddr(io), peeraddrstr));
    printf("< %s\n", buf);
    // echo
    printf("> %s\n", buf);
    hio_write(io, buf, readbytes);
}

void on_accept(hio_t* io) {
    printf("on_accept connfd=%d\n", hio_fd(io));
    char localaddrstr[SOCKADDR_STRLEN] = {0};
    char peeraddrstr[SOCKADDR_STRLEN] = {0};
    printf("accept connfd=%d [%s] <= [%s]\n", hio_fd(io),
            SOCKADDR_STR(hio_localaddr(io), localaddrstr),
            SOCKADDR_STR(hio_peeraddr(io), peeraddrstr));

    hio_setcb_close(io, on_close);
    hio_setcb_read(io, on_recv);
    hio_set_readbuf(io, recvbuf, RECV_BUFSIZE);
    hio_read(io);
}

int main(int argc, char** argv) {
    if (argc < 2) {
        printf("Usage: cmd port\n");
        return -10;
    }
    int port = atoi(argv[1]);

    hloop_t* loop = hloop_new(0);
    hio_t* listenio = create_tcp_server(loop, "0.0.0.0", port, on_accept);
    if (listenio == NULL) {
        return -20;
    }
    printf("listenfd=%d\n", hio_fd(listenio));
    hloop_run(loop);
    hloop_free(&loop);
    return 0;
}
```
```
make tcp udp nc
bin/tcp 1111
bin/nc 1111

bin/udp 2222
bin/nc -u 2222
```

## BUILD

### lib
- make libhv
- make install

### examples
- make test # master-workers model
- make timer # timer add/del/reset
- make loop # event-loop(include idle, timer, io)
- make tcp  # tcp server
- make udp  # udp server
- make nc   # network client
- make nmap # host discovery
- make httpd # http server
- make curl # http client

### unittest
- make unittest

### compile options
#### compile with print debug info
- make DEFINES=PRINT_DEBUG

#### compile WITH_OPENSSL
- make DEFINES=WITH_OPENSSL

#### compile WITH_CURL
- make DEFINES="WITH_CURL CURL_STATICLIB"

#### compile WITH_NGHTTP2
- make DEFINES=WITH_NGHTTP2

#### other options
- ENABLE_IPV6
- WITH_WINDUMP
- USE_MULTIMAP

## Module

### data-structure
- array.h:       动态数组
- list.h:        链表
- queue.h:       队列
- heap.h:        堆

### base
- hplatform.h:   平台相关宏
- hdef.h:        宏定义
- hversion.h:    版本
- hbase.h:       基本接口
- hsysinfo.h:    系统信息
- hproc.h:       子进程/线程类
- hmath.h:       math扩展函数
- htime.h:       时间
- herr.h:        错误码
- hlog.h:        日志
- hmutex.h：     同步锁
- hthread.h：    线程
- hsocket.h:     socket操作
- hbuf.h:        缓存类
- hurl.h:        URL转义
- hgui.h:        gui相关定义
- hstring.h:     字符串
- hvar.h:        var变量
- hobj.h:        对象基类
- hfile.h:       文件类
- hdir.h:        ls实现
- hscope.h:      作用域RAII机制
- hthreadpool.h: 线程池
- hobjectpool.h: 对象池

### utils
- hmain.h:       main_ctx: arg env
- hendian.h:     大小端
- ifconfig.h:    ifconfig实现
- iniparser.h:   ini解析
- singleton.h:   单例模式
- md5.h
- base64.h
- json.hpp

### event
- hloop.h:       事件循环

#### iowatcher
- EVENT_SELECT
- EVENT_POLL
- EVENT_EPOLL   (linux only)
- EVENT_KQUEUE  (mac/bsd)
- EVENT_IOCP    (windows only)

### http
- http_client.h: http客户端
- HttpServer.h:  http服务端

### other

- hv.h：         总头文件
- Makefile.in:   通用Makefile模板
- main.cpp.tmpl: 通用main.cpp模板

