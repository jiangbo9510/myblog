---
title: 关于转义字符
date: 2019-05-03
tag: char
categories: 2019

---


如下图所示：
当没有使用转移字符的时候，\n的长度为1，使用strchr是可以找到\n的

<!-- more -->

![](http://ww1.sinaimg.cn/large/ea3ade00gy1g2o87kp7lqj20mo0amt8z.jpg)



但是，当使用了转移字符后，如下图所示：
\\n的长度变成了2，并且也无法找到\n字符了
![](http://ww1.sinaimg.cn/large/ea3ade00gy1g2o88regjbj20ly0aqq37.jpg)

由此可见，\n经过转义字符\的变化后，从ASCII表中的一个独立字符，变成了两个字符

