---
author: admin
comments: true
date: 2011-02-17 03:11:06+00:00
layout: post
slug: trap-in-syn-attack-protect
title: Win2003Server的SYN Attack Protect陷阱
wordpress_id: 381
categories:
- 默认
tags:
- socket
- synattackprotect
- TCPIP
- win2003server
---



发现这个陷阱源于一个自己写的HTTP服务端。因为项目里要用到支持GET请求就能满足需要的HTTP服务端，图方便就用内部封的异步Socket类实现了这么个简单的服务端。没想到，就是这个简单的HTTP服务，还是出现了几次莫名其妙的问题。第一次出现在刚完成代码功能测试时，对HTTP的一知半解导致查了几天之后才发现原来是没处理HTTP头里的Keep-Alive。

最近又碰见另一个百思不得其解铃还需系铃人人得而诛之的问题，测试服务端性能时，用Apache的AB压服务器，设置的并发数稍大就只能完成一部分请求，AB的提示是“远程主机主动关闭连接”。具体来说就是，当AB参数为请求数1000，并发数200的时候，服务端能够正常完成测试；增加并发数到300，则服务端只能完成1000个请求中的300多个，客户端抓包观察到大量请求在TCP三次握手完成后立刻被RST。

用自己封装的异步Socket类写个测试客户端，循环连接服务端，产生大量连接，同样连接数也只能到300多个，其余的大量连接得到类型为WSAENETRESET的socket错误。

用来实现HTTP服务端的异步Socket类没有什么特别操作。逻辑是这样的：



	
  * 服务端首先产生一个Socket句柄（废话...），转为异步模式后监听端口，等待客户端连接。

	
  * 一旦有客户端连接，服务端调用accept得到新的Socket，加入到一个Socket集合进行统一处理。

	
  * 每次统一处理时，查询所有Socket的事件，对不同的事件进行不同的处理。

	
  * 服务端对所有的Socket句柄不主动执行关闭操作，除非调用recv时得到的长度为0，即认为是客户端主动关闭，随后服务端进入被动关闭。


应用开发出现问题的第一反应是检查自己的程序。首先怀疑是逻辑不严密，在某特殊情况下Reset客户端连接。在服务端每个可能关闭socket的地方增加日志，再用AB压一次观察输出。但服务端日志没有记录到任何异常的关闭操作，问题似乎不是服务端关闭连接导致的。

既然没有异常关闭socket的情况，那应该是其他地方出问题。考虑listen函数的backlog参数，会不会是这里的原因呢？

listen函数原型：

    
    int listen( __in  SOCKET s,  __in  int backlog );


来看backlog参数在MSDN的说明


> The maximum length of the queue of pending connections


再往下看，MSDN中对listen函数有一段这样的描述


> If a connection request arrives and the queue is full, the client will receive an error with an indication of WSAECONNREFUSED.


就是说，如果客户端连接数过大，超过调用listen函数设置的backlog队列大小，新的连接会收到WSAECONNREFUSED的socket错误。但观察到的现象是客户端socket错误为WSAENETRESET，不符合文档的描述。

过了listen函数再往下看，代码进入日志能够记录的范围。上面说过，服务端没有异常关闭的操作。到这里仍然没找到原因。目前为止，最有可能的情况就是瞬间的连接数过多导致backlog满。就算是这个原因，客户端收到的也应该是WSAECONNREFUSED错误而不是观察到的WSAENETRESET错误，绕进死胡同了。就这个问题在[stackoverflow](http://stackoverflow.com/)发的询问帖，回复里也有人说可能是backlog满导致客户端无法连接，但却无法解释为什么收到的socket错误不是MSDN描述的WSAECONNREFUSED。

问题就这样被搁置下来，直到三天后我忙完手头的事情，用部分关键词google，得到了某个错误现象匹配度很高的搜索结果。[这里](http://www.tech-archive.net/Archive/Development/microsoft.public.win32.programmer.networks/2005-10/msg00313.html)是原帖，截取一段简单描述


> We started getting problems when customers implemented SP1 on their Windows Server 2003
boxes. We'd see clients who thought they had successfully connected, but who would get a
RESET back from the server immediately after the 3-way handshake (SYN, SYN-ACK, ACK). This
seems *completely broken* to me.


啊哈，跟我们的问题看起来一样！他认为是Windows 2003 Server默认开启的[SYN攻击保护](http://technet.microsoft.com/en-us/library/cc781167(WS.10).aspx)开关在作怪，猜想可能是SYN攻击保护机制的实现影响到Winsock的accept函数行为。正常的行为是，当backlog队列满时，三次握手中第一个SYN包就会导致服务端响应RST+ACK拒绝客户端的连接。但SYN保护机制起作用后，这个行为变成直接接受新的连接再去检查backlog队列，当发现backlog队列满时丢弃此连接。服务端此时要干掉连接就只能在已完成三次握手的连接上发送一个RST，于是客户端出现莫名其妙的WSAENETRESET错误。

这是他的问题以及猜测，能不能解决我们的问题，还要实验一下。打开注册表才发现，我们的服务器上没有他描述的HKLM\System\CurrentControlSet\Services\Tcpip\Parameters\SynAttackProtect这一项，没关系，没有我们可以创造一个，设置键值为0，重启机器（万恶的重启！）。此时再用AB去压服务端，在客户端果然抓到的RST+ACK包。用自己写的客户端测试，socket错误也由之前百思不得其解的WSAENETRESET变成WSAECONNREFUSED。

问题原因到这里基本确认，由于我们的服务端实现在接受客户端连接时速度不够快，用AB压服务端时的大量连接填满backlog队列，这些连接同时也触发服务器默认开启的SYN攻击保护机制，操作系统认为受到SYN攻击，后续的其他连接于是被重置。关于这个问题，[这里](http://blogs.msdn.com/b/wndp/archive/2005/12/09/502256.aspx)也有提及。这种跟文档描述不一致的行为害苦了广大程序员，在别人2005年就受过苦之后，2010年我这个倒霉蛋又再次遇上。不过奇怪的是，网上搜索不到多少关于这个问题的中文资料，英文倒是又不少人提过。

之前在跟项目组同事的讨论原因时，也考虑过系统或者路由在应用程序之下断开连接的可能性，但是由于对操作系统环境不熟悉，不知道存在这么一个SynAttackProtect机制，影响到问题原因的确认，算是一个教训吧。虽然存在文档描述与实际行为不一致的原因，但对程序运行平台的熟悉程度还是直接影响了定位问题的速度。

顺便说一句，测试中发现，Windows 2003 Server SP2里的backlog最大值似乎是200，不过没有查过官方文档确认，纯属推测。


