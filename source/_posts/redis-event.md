---
title: Redis 客户端与服务器连接流程实例
tags:
  - redis
date: 2018-12-17 16:26:42
---


如下图所示，为redis 客户端连接服务器完整的流程。

![redis 客户端与服务器连接事件](https://s1.ax1x.com/2018/12/17/F0833n.png)

1.首先在redis sever 启动的时候，会把AE_READABLE事件和acceptTcpHandler方法绑定，向eventLoop注册。

```c
if (server.ipfd > 0 && aeCreateFileEvent(server.el, server.ipfd, AE_READABLE, acceptTcpHandler, NULL) == AE_ERR)
        redisPanic("Unrecoverable error creating server.ipfd file event.");
```

2.当client连接server时，会触发redis sever的AE_READABLE事件为就绪状态。

3.当AE_READABLE事件为就绪状态时，会在aeMain中对其进行处理，并执行绑定的acceptTcpHandler方法。在acceptTcpHandler方法中，会创建client实例，并将client的AE_READABLE事件和readQueryFromClient方法绑定，向eventLoop注册。

```c
redisClient *createClient(int fd) {
    ...
    if (aeCreateFileEvent(server.el, fd, AE_READABLE, readQueryFromClient, c) == AE_ERR) {
            close(fd);
            zfree(c);
            return NULL;
    }
    ...
}
```

4.client向server想要执行的命令，触发client的AE_READABLE事件变为就绪状态。

5.同理，在aeMain中对AE_READABLE变为就绪状态的事件进行处理。执行绑定的readQueryFromClient方法，并执行相应的命令。在命令执行过后准备发送结果给client之前，会把client的AE_WRITEABLE事件和sendReplyToClient方法绑定， 向eventLoop注册，同时发送命令，触发AE_WRITEABLE事件。

```c
int prepareClientToWrite(redisClient *c) {
    // ...
    if (c->bufpos == 0 && listLength(c->reply) == 0 && (c->replstate == REDIS_REPL_NONE || c->replstate == REDIS_REPL_ONLINE) &&
        aeCreateFileEvent(server.el, c->fd, AE_WRITABLE, sendReplyToClient, c) == AE_ERR)
        return REDIS_ERR;
    return REDIS_OK;
}
```

6.在aeMain中对AE_WRITEABLE的事件进行处理，执行绑定的sendReplyToClient方法，把命令发送给client，同时删除向eventLoop注册的AE_WRITEABLE事件。

```c
void sendReplyToClient(aeEventLoop *el, int fd, void *privdata, int mask) {
    // ...
    // 回复全部发送完毕，删除事件处理器
       if (c->bufpos == 0 && listLength(c->reply) == 0) {
           c->sentlen = 0;
           aeDeleteFileEvent(server.el, c->fd, AE_WRITABLE);

           // 如果状态为“回复完毕之后关闭”，那么关闭客户端
           if (c->flags & REDIS_CLOSE_AFTER_REPLY) freeClient(c);
       }
}
```

#### reference

* https://redisbook.readthedocs.io/en/latest/internal/ae.html
