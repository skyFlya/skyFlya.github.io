---
title: JS number 精度丢失问题
tags:
  - JS
categories: JS
thumbnail: 'https://s1.ax1x.com/2022/03/14/bLrVJK.md.png'
date: 2022-05-25 15:39:11
---
> 数值位大引发的精准度问题

<!--more-->
# **如果数值超过16位，就用String类型！否则序列化后数值会不准！**

# 例子：
用户反馈自己的用户ID，需要做一些业务操作，结果发现通过用户截图的用户ID，并不能在数据库里找到该用户
后经发现，客户端 JS 受制于 精准度 的问题，一旦长度超过16位，就会导致序列化后不准


服务器发送的数据，userid， 浏览器实际接收到的数据
![bWKCWV.png](https://s1.ax1x.com/2022/05/25/XkP2b8.png)

客户端序列化后的数据，可以看到数值已经变了
![bWKCWV.png](https://s1.ax1x.com/2022/05/25/XkPfUg.png)

# 解决方案：
超过16位的数值，对应服务器long类型，下发一律使用string。