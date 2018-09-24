---
title: 后台web框架
date: 2018-09-12 23:50:01
tags:
  - web
  - business logic
---

首先用golang原生的框架实现一些功能。比如说web router和orm的功能。

web 路由，静态文件，模板，cookies

## 官方给的golang web hello world

```golang

package main

import (
	"io"
	"net/http"
	"log"
)

// hello world, the web server
func HelloServer(w http.ResponseWriter, req *http.Request) {
	io.WriteString(w, "hello, world!\n")
}

func main() {
	http.HandleFunc("/hello", HelloServer)
	log.Fatal(http.ListenAndServe(":12345", nil))
}
```

### 首先来说使用官方web框架所遇到的一些问题。或者说是一些不好用的地方。

### golang处理一次HTTP请求的流程：

#### 注册路由相关

```golang

type muxEntry struct {
	h       Handler
	pattern string
}

type ServeMux struct {
	mu    sync.RWMutex        
	m     map[string]muxEntry
	hosts bool                
}

var defaultServeMux ServeMux
var DefaultServeMux = &defaultServeMux

// HandleFunc registers the handler function for the given pattern in the DefaultServeMux.
func HandleFunc(pattern string, handler func(ResponseWriter, *Request)) {
	DefaultServeMux.HandleFunc(pattern, handler)
}
```

首先是使用http.HandleFunc来注册路由，直接调用的话是使用默认的DefaultServeMux。其中还有一些细节，比如说不以'/'开头的路径，会被设置为包含了hosts的；注册同名的路径名会panic等。

#### 服务器启动，开启监听请求

```golang
log.Fatal(http.ListenAndServe(":12345", nil))

func (srv *Server) Serve(l net.Listener) error {
    defer l.Close()

    ...

    for {
        // 等待接受请求
        rw, e := l.Accept()
        if e != nil {
            ...
        }
        tempDelay = 0
        // 为本次请求创建一个Connection
        c := srv.newConn(rw)
        c.setState(c.rwc, StateNew)
        // 启动一个goroutine来对本次请求进行处理
        // ctx中有两个值，一个是server，一个是connection
        go c.serve(ctx)
    }
}
```

#### 启动goroutine对请求进行处理

```golang

func (c *conn) serve(ctx context.Context) {
    ...

    c.r = &connReader{r: c.rwc}
    c.bufr = newBufioReader(c.r)
    c.bufw = newBufioWriterSize(checkConnErrorWriter{c}, 4<<10)

    ctx, cancelCtx := context.WithCancel(ctx)
    defer cancelCtx()

    // TODO @lmj 这里为啥是一个for循环呢？
    for {
        // 从connecton中读取请求
        w, err := c.readRequest(ctx)
        ...
        // 对读取的请求开始进行处理
        serverHandler{c.server}.ServeHTTP(w, w.req)
        w.cancelCtx()
        // 本次请求处理结束
        w.finishRequest()
        if !w.shouldReuseConnection() {
            if w.requestBodyLimitHit || w.closedRequestBodyEarly() {
                c.closeWriteAndWait()
            }
            return
        }
        c.setState(c.rwc, StateIdle)
    }
}
```

#### 查找对应的hanlder对请求进行处理

```golang
// ServeHTTP 会把请求分发给匹配的handler
func (mux *ServeMux) ServeHTTP(w ResponseWriter, r *Request) {
	if r.RequestURI == "*" {
	  if r.ProtoAtLeast(1, 1) {
		    	w.Header().Set("Connection", "close")
		}
		w.WriteHeader(StatusBadRequest)
		return
	}
	// 获取处理本次请求的handler
	h, _ := mux.Handler(r)
	// 真正开始处理本次请求
	h.ServeHTTP(w, r)
}
```

如果没有设置自己的路由，就会使用的默认的 defaultServeMux；这个里面可以决定哪些url分配给哪些handler来进行处理。

上面的内容可以单独拆分成一部分，叫做一次HTTP请求的处理流程。


然后说明一下这个web框架需要解决问题。

最后介绍web框架。

这里还有一些可以说，比如说log框架。
