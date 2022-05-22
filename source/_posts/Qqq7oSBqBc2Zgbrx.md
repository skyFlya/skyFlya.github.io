---
title: CS理论 - 计算机网络
tags:
  - 计算机基础
  - 网络
categories: CS理论
updated: 1635687032000
thumbnail: 'https://s1.ax1x.com/2022/03/14/bL0Ei8.md.png'
date: 2021-10-31 21:31:00
---
## TCP/IP

> 两台计算机之间的通讯是通过TCP/IP协议在因特网上进行的

TCP：Transmission Control Protocol 传输控制协议, 用于应用程序之间的通信
IP：Internet Protocol 网际协议, 用于计算机之间的通信
<!-- more -->
#### 分层

> 因为网络具有不稳定性,要保证通信正常

![5qccSf.png](https://z3.ax1x.com/2021/10/28/5qccSf.png)


#### TCP连接

> 通信双方建立确认「可以通信」，不会将对方的消息丢弃，即为「建立连接」

##### 连接建立(三次握手)

![5LRCDS.png](https://z3.ax1x.com/2021/10/28/5LRCDS.png)


##### 连接关闭

![5LRGCR.png](https://z3.ax1x.com/2021/10/28/5LRGCR.png)


##### 长连接

> 强制不让连接的通道关闭

- TCP短连接
	- client向server发起连接请求，server接到请求，然后双方建立连接。client向server 发送消息，server回应client，然后一次读写就完成了
	- 一般的server不会回复完client后立即关闭连接的, 所以一般是client先发起close操作
- TCP长连接
	- client向server发起连接，server接受client连接，双方建立连接。Client与server完成一次读写之后，它们之间的连接并不会主动关闭，后续的读写操作会继续使用这个连接

怎样实现长连接？
⼼跳。即在一定间隔时间内，使⽤TCP连接发送超短无意义消息来让网关不能将⾃己定义为「空闲连接」，从而防止网关将⾃己的连接关闭

## HTTP

> Hyper Text Transfer Protocol（超文本传输协议）缩写,基于 TCP/IP 通信协议来传输数据,应用层协议

-   简单快速：只需传送请求方法和路径。请求方法常用的有GET、HEAD、POST。由于简单，因而通信速度很快。

-   灵活：HTTP 允许传输任意类型的数据对象，传输的类型由 Content-Type 标记。

-   无连接：限制每次连接只处理一个请求。服务器处理完客户的请求，并收到客户的应答后，即断开连接，传输时间。
    
-   无状态：指协议对于事务处理没有记忆能力，后续处理需要前面的信息，必须重传，导致每次连接传送的数据量增大。但不需要先前信息时应答快。

### 报文组成

请求报文:

-   请求行：包括请求方法、URL、协议/版本

-   请求头(Request Header)   

-   请求正文
    
```json
GET / HTTP1.1             
Host:https://wanandroid.com/
User-Agent:Mozilla/5.0 (Linux; Android 6.0; Nexus 5 Build/MRA58N) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/86.0.4240.198 Mobile Safari/537.36
Content-Type:application/json;charset=UTF-8
Content-Length:40
Connection:Keep-Alive

name: "test"
```

响应报文:

-   状态行    

-   响应头    

-   响应正文
    
```json
HTTP/1.1 200 OK
Content-Type: application/json;charset=UTF-8
Date: Wed, 03 Feb 2020 09:47:32 GMT

{"data":[{"children":[],"courseId":1
```

### 原理

-   基于 TCP/IP 通信协议来传递数据的协议
-   用于 B/S 架构
-   默认端口 80

![bgpj9x.png](https://s1.ax1x.com/2022/03/08/bgpj9x.png)


### 发展

### HTTP 1.0

-   请求与响应支持 HTTP 头，响应含状态行，增加了状态码
-   支持 HEAD，POST 方法
-   支持传输 HTML 文件以外其他类型的内容

缺点：

-   非持久连接：客户端必须为每一个待请求的对象建立并维护一个新的连接，短连接增加了网络传输的负担

#### HTTP 1.1

-   支持长连接
-   在HTTP1.0的基础上引入了更多的缓存控制策略
-   引入了请求范围设置，优化了带宽
-   在错误通知管理中新增了错误状态响应码
-   增加了Host头处理，可以传递主机名（hostname）

#### HTTPS

-   HTTPS 运行在安全套接字协议(Secure Sockets Layer，SSL )或传输层安全协议（Transport Layer Security，TLS）之上，所有在TCP中传输的内容都需要经过加密。
-   连接方式不同，HTTP的端口是 80，HTTPS的端口是 443.
-   HTTPS 可以有效防止运营商劫持。

#### HTTP 1.x优化（SPDY）

SPDY ：在 HTTP 之前做了一层会话层，为了达到减少页面加载时间的目标，SPDY 引入了一个新的二进制分帧数据层，以实现优先次序、最小化及消除不必要的网络延迟，目的是更有效地利用底层 TCP 连接。

-   多路复用，为多路复用设立了请求优先级
-   对 header 部分进行了压缩
-   引入了 HTTPS 加密传输
-   客户端可以在缓存中取到之前请求的内容

#### HTTP 2.0

-   使用二进制分帧层：在应用层与传输层之间增加一个二进制分帧层
-   多路复用
-   服务端推送：logo、CSS 文件直接推给客户端
-   数据流优先级：对数据流可以设置优先值，比如设置先传 css 文件等
-   头部压缩
-   支持明文传输，HTTP 1.X 使用 SSL/TLS 加密传输。

![bg9bM8.png](https://s1.ax1x.com/2022/03/08/bg9bM8.png)

#### HTTP 3.0（QUIC）

QUIC (Quick UDP Internet Connections)，快速 UDP 互联网连接，基于UDP 协议的。

- 线头阻塞(HOL)问题的解决更为彻底
   基于TCP的HTTP/2，尽管从逻辑上来说，不同的流之间相互独立，不会相互影响，但在实际传输方面，数据还是要一帧一帧的发送和接收，一旦某一个流的数据有丢包，则同样会阻塞在它之后传输的流数据传输。而基于 UDP 的QUIC 协议则可以更为彻底地解决这样的问题，让不同的流之间真正的实现相互独立传输，互不干扰。

- 切换网络时的连接保持
  当前移动端的应用环境，用户的网络可能会经常切换，比如从办公室或家里出门，WiFi断开，网络切换为 3G 或 4G。基于TCP的协议，由于切换网络之后，IP会改变，因而之前的连接不可能继续保持。而基于 UD P的 QUIC 协议，则可以内建与 TCP 中不同的连接标识方法，从而在网络完成切换之后，恢复之前与服务器的连接。

## HTTPS

>  是以安全为目标的 HTTP 通道(加密通信的HTTP)，其实就是 **HTTP+SSL/TLS**

#### 工作原理

> 在客户端和服务器之间协商出一套对称密钥，每次发送消息之前将内容加密，收到之后解密，达到内容的加密传输

##### 为什么不直接用非对称加密?
非对称加密由于使用了复杂的数学原理，因此计算相当复杂，如果完全使用非对称加密来加密通信内容，会严重影响网络通信的性能


#### HTTPS流程  

![bgCnRx.png](https://s1.ax1x.com/2022/03/08/bgCnRx.png)

1. 客户端请求建立TLS连接
2. 服务器发回证书
3. 客户端验证服务器证书
4. 客户端信任服务器后，和服务器协商对称密钥
5. 使用对称密钥开始通信



## 授权

登录：身份认证，确认你是你的过程。  
授权：由身份或持有的令牌确认享有某些权限（例如获取用户信息）。登录过程实质上的目的也是为了确认权限

### Cookie

> 使用Cookie管理登录状态

![5qrP4f.png](https://z3.ax1x.com/2021/10/28/5qrP4f.png)

![5qrmbn.png](https://z3.ax1x.com/2021/10/28/5qrmbn.png)

#### Cookie工作机制
1. 服务器需要客户端保存的内容，放在Set-Cookie的headers里返回，客户端会自动保存。
2. 客户端保存的Cookies，会在之后的所有请求里都携带进Cookie header里发回给服务器。
3. 客户端保存Cookie是按照服务器域名来分类的，例如shop.com发回的Cookie保存下来以后，在之后向games.com的请求中并不会被携带。
4. .客户端保存的Cookie在超时后会被删除，没有设置超时时间的Cookie（称作Session Cookie）在浏览器关闭后就会自动删除，另外，服务器也可以主动删除还未过期的客户端Cookies
5. Cookie是由服务端管理的，客户端只是被动接受。


### Authorization Header

> 两种主流方式 Basic 和 Beare

#### Basic

```dart
1.格式：Authorization: Basic<username:password(Base64)>
2.Basic认证过程
1.浏览器发送请求到服务器
                GET / HTTP /1.1
                Host:xxx.xxx.com
2.服务端发送验证请求401
                HTTP/1.1 401 Unauthorised
                Server: bfe/1.0.8.18
                WWW-Authenticate: Basic realm="XXXXX.com"
                Content-Type: text/html; charset=utf-8
3.客户端收到401返回值后，将自动弹出登录窗口，等待用户输入用户名，密码
4.将用户名密码进行Base64编码后发送给服务端进行验证
                GET / HTTP/1.1
                Host: xxx.xxx.xxx.com
                Authorization: Basic xxxxxxxxxxxxxxxxxxxxxxxxxxxx
5.服务端取出Authorization中头信息，并与数据库进行比对，如果合法则返回200，不合法，则返回401。
```

#### Bearer

```objectivec
1.格式：Authorization: Bearer <bearer token>
2.bearer token 的获取方式：通过OAuth2的授权流程
3.OAuth2的流程
            1.第三方网站向授权网站申请授权合作，拿到client id和client secret
            2.用户在使用第三方网站的时候，点击通过授权，第三方网站将跳转授权网站，并将clientid传递给授权网站。
            4.授权方网站根据clientid，将第三方网站的信息和第三方网站需要的用户权限展示给用户，询问用户是否同意授权。
            5.当用户点击同意授权，授权方网站返回第三方网站，并传入Authorization code作为用户认可的凭证。
            6.第三方网站将Authorization code上送给自己的服务器，服务器将Authorization code跟自己服务器端存储的client secret发送给授权方服务器，授权方服务器通过验证后，返回给access token，整个OAuth2流程结束。
            7.在整个OAuth流程结束后，第三方网站服务器可以试用access token 作用用户授权的token，用此来向授权方网站请求获取用户信息等操作。
        
4.问题：
            为什么OAuth认证要引入Authorization code，并且需要申请授权的第三方将Authorization code传给第三方服务器，并且通过第三方服务器将Authorization code传递给授权方服务器。然后再获取授权方服务器的access token。这样做的目的是为了通信安全，因为OAuth不强制使用HTTPS，因此需要保证通信路径中存在窃听者的时候，还能保证足够的安全。
            
5.第三方APP通过微信登录的流程：
            这是一个标准的OAuth2的流程。
            1.第三方APP向腾讯方申请合作，拿到client id 和client secret
            2.当用户在第三方APP上需要微信登录的时候，第三方APP将使用微信SDK打开微信授权页面，并且传入client id作为自己授权id。
            3.微信拿到第三方app的client id后，提交微信后台，验证成功，则返回给第三方app Authorization code。
            4.第三方app拿到Authorization code后，将Authorization code传递给自己的服务器，第三方app的服务端将Authorization code 与 client secret 传递给微信后台，微信后台验证后返回access token。
            5.第三方app后台通过access token 想微信后台请求获取用户信息，微信验证通过后，则返回用户信息。
            6.服务器接受到用户信息后，在自己的数据库中建立一个账户，并将从微信获取的用户信息填入数据库中。并创建用户id，并将此id与微信id做好关联。
            7.当第三方app后台创建好用户后，想客户端的请求发出响应，并回传会刚刚创建的用户信息。
            8.客户端响应，获取用户信息，登录成功。
            
6.在自家APP中使用Bearer token
            部分app会在api设计中，将登录和授权设计出类似于OAuth2的过程，他会简化掉Authorization code的概念。既直接在接口请求成功后，返回access token，然后再之后的客户端的请求中，使用这个access token 作为Bearer token进行用户操作。
            
7.Refresh token
            Access token 都会有失效时间，当他失效后，第三方app的服务端会通过refresh token 接口传入refresh token 来获取新的access token。这样的的原因是安全，因为refresh token是放置在服务端的，即使access token被窃取，他也是有失效时间的。
```


## Refer
[登录授权、TCP/IP、HTTPS和代理](https://www.jianshu.com/p/2a890d952461)
[编码、加密、Hash、TCP/IP、HTTPS](https://blog.csdn.net/kimlllll/article/details/103041225)
[HencoderPlus/04-TCPIP和HTTPS](https://github.com/hsicen/HencoderPlus/blob/master/note/04-TCPIP%E5%92%8CHTTPS.md)