---
title: redis的网络
date: 2019-05-03
tag: redis
categories: 2019

---


- 第一步：创建epoll

![](http://ww1.sinaimg.cn/large/ea3ade00gy1g2nxgq9fzzj20ig0icgnn.jpg)

如上所示，创建epoll，并放入eventLoop中。

<!-- more -->

- 第二步：监听端口

![](http://ww1.sinaimg.cn/large/ea3ade00gy1g2ny6bohf3j20fu03w74q.jpg)

因为一台机器可能存在多个ip，甚至由于docker，导致内部又分配了ip。故，redis会针对所有的ip，对同一个端口监听（默认6379）


![](http://ww1.sinaimg.cn/large/ea3ade00gy1g2ny83sfvjj20g5032weq.jpg)

上图就是在redisServer中，存放各个ip地址的结构


- 第三步：注册函数

![](http://ww1.sinaimg.cn/large/ea3ade00gy1g2nygzmkz6j20hh07zdgw.jpg)

将第二步监听的fd绑定处理函数后，加入epoll中进行监听

其中acceptTcpHandler就是用于处理新连接的函数

- 第四步：建立连接

![](http://ww1.sinaimg.cn/large/ea3ade00gy1g2nynbkvxgj20fn0f576a.jpg)

![](http://ww1.sinaimg.cn/large/ea3ade00gy1g2nz1w1xdjj20ef06aq3i.jpg)

![](http://ww1.sinaimg.cn/large/ea3ade00gy1g2nz40jg2dj20c605774n.jpg)

上述三个图为一体，建立连接后，redis会在本地创建redisClient，将新连接的fd加入epoll监听。

readQueryFromClient函数就是处理客户端命令的函数


第五步：接收命令返回结果

    typedef struct redisClient {

    // 套接字描述符
    int fd;

    // 当前正在使用的数据库
    redisDb *db;

    // 查询缓冲区
    sds querybuf;

    // 查询缓冲区长度峰值
    size_t querybuf_peak;   /* Recent (100ms or more) peak of querybuf size */

    // 参数数量
    int argc;

    // 参数对象数组
    robj **argv;

    // 记录被客户端执行的命令
    struct redisCommand *cmd, *lastcmd;

    // 请求的类型：内联命令还是多条命令
    int reqtype;

    // 剩余未读取的命令内容数量
    int multibulklen;       /* number of multi bulk arguments left to read */

    // 命令内容的长度
    long bulklen;           /* length of bulk argument in multi bulk request */

    // 回复链表
    list *reply;

    // 回复链表中对象的总大小
    unsigned long reply_bytes; /* Tot bytes of objects in reply list */

    // 已发送字节，处理 short write 用
    int sentlen;            /* Amount of bytes already sent in the current
                               buffer or object being sent. */
    /* Response buffer */
    // 回复偏移量
    int bufpos;
    // 回复缓冲区
    char buf[REDIS_REPLY_CHUNK_BYTES];

	} redisClient;

上述结构体是客户端的构成，其中包括查询和结果的缓冲


    以 set msg hello 为例，介绍redis处理命令的全过程


0、redis的请求都是末尾都是\n结尾，故检查报文的完整程度的时候，会根据最后有没有\n判断


    /* Search for end of line */
    newline = strchr(c->querybuf,'\n');

    /* Nothing to do without a \r\n */
    // 收到的查询内容不符合协议格式，出错
    if (newline == NULL) {
        if (sdslen(c->querybuf) > REDIS_INLINE_MAX_SIZE) {
            addReplyError(c,"Protocol error: too big inline request");
            setProtocolError(c,0);
        }
        return REDIS_ERR;//这里的出错返回不做任何处理，等待下一次epoll触发继续读即可
    }


1、解析命令，将命令解析用空格隔开。存入redisClient的argv和argc中

2、解析完毕后，调用processCommand函数，根据set命令查找对应的处理函数setCommand，存入redisClient的cmd中，并检查参数数量等

3、处命令完成后，由

    // 设置成功，向客户端发送回复
    // 回复的内容由 ok_reply 决定
    addReply(c, ok_reply ? ok_reply : shared.ok);

函数向客户端返回结果

4、addReply的实现

    aeCreateFileEvent(server.el, c->fd, AE_WRITABLE,
        sendReplyToClient, c) == AE_ERR) //通过该函数注册返回的结果
    .....

    // 复制内容到redisClient的 c->buf 里面
    memcpy(c->buf+c->bufpos,s,len);
    c->bufpos+=len;

这也就决定了，同一个连接一次只能执行一个命令
