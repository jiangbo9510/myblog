---
title: redis的内存优化
date: 2019-05-03
tag: redis
categories: 2019

---

redis一个大list拆分为N个小list用压缩列表保存，可以更省内存。

	列表对象保存的所有字符串元素的长度都小于64字节；
	列表对象保存的元素数量小于512个；

下面将举例说明。

<!-- more -->

## 情况1、一个key，有很多64字节以内的value ##


例如：

![](http://ww1.sinaimg.cn/large/ea3ade00gy1g2obm7jh6xj206601fq2p.jpg)

	只有一个key

![](http://ww1.sinaimg.cn/large/ea3ade00gy1g2obncrsd5j208001a0si.jpg)

	该key对应的list有300w个value

![](http://ww1.sinaimg.cn/large/ea3ade00gy1g2obqrgqakj209s057wee.jpg)

	每个value都是小于64个字节的

![](http://ww1.sinaimg.cn/large/ea3ade00gy1g2obocvpl8j206q047jr9.jpg)

	在只有一个300w个value的key的情况下，内存占用约为263m


## 情况2：将value分散到不同的key ##

将同样的3434114个value拆分到6800个key中，每个key存放500个value。

数据如下：

![](http://ww1.sinaimg.cn/large/ea3ade00gy1g2ocbgf1rrj2068017mwx.jpg)
	
	有6800+个key

![](http://ww1.sinaimg.cn/large/ea3ade00gy1g2occrgj6fj20cv0b0t8v.jpg)

	value和情况1是一致的

![](http://ww1.sinaimg.cn/large/ea3ade00gy1g2ocbv77kej207i04fq2t.jpg)

	消耗的内存只有55m

## 原因分析 ##

情况1：

列表的node的双向指针每个8个字节,指向redisObject的字符串的指针8个字节

redisObeject消耗内存24个字节

字符串以sds的方式存储，每个16个字节（含2*sizeof(int)+真实字符串长度）。

![](http://gameweb-img.qq.com/gad/20170727/519ce13cec1d52deda9c1ce5489298b4.1501147736.png)

共消耗内存 = 350w*(24+24+16) = 224m


情况2：

已ziplist的形式存储，

只消耗内存：

6800*(500*(12+4) + 12) = 54m
其中6800位key的数量，500位每个key对应的value的数量，12位字符串长度，4位int大小，记录字符串长度，12位每个ziplist的消耗的内存
