---
title: redis的RDB简介
date: 2019/04/14
tag: redis
categories: 2019
---


**RDB的全称是Redis Database File，redis的RDB刷盘为异步刷盘，配置N秒内产生了M次更新操作则触发刷盘。
redis内部会每100ms进行一次检测。判断是否需要刷盘。**

刷盘的配置：
    
    save 900 1   //表示900s内有一次修改
    save 300 10  //表示300s内有10次修改
    save 60 10000 //表示60s内有10000次修改

满足上述条件后就会刷盘    


<!-- more -->

1、redis的save命令会导致阻塞，无法执行其他的命令。

2、redis的bgsave命令会开启新的进程来进行备份，由于新的进程的数据空间会和父进程的独立，故相当于刷盘的数据是这一刻的快照。

3、redis的rdb文件在执行redis-serverD目录位置下，可以用这个命令来寻找

    `find / -name dump.rdb`

4、

![](http://ww1.sinaimg.cn/large/ea3ade00ly1g22key0jnbj20ev01vdfm.jpg)

    简要分析rdb文件：
    REDIS 表示这个文件是redis的rdb文件。
    0006 表示版本号
    379 表示下一个字节是数据库编号
    0 表示是第0个db
    0 表示保存的类型的REDIS_DB_TYPE_STRING
    003 表示下一个字符串长度为3个字符
    msg 表示是key的字符
    005 表示下一个字符串长度为5个字符
    hello 表示value的字符
    337 表示的是EOF常量
    剩余的8个字符表示的是文件校验，用于判断文件是否完整
