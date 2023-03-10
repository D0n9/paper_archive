# https流量解密方案综述 - FreeBuf网络安全行业门户
一、https发展历史
-----------

### http和https的发展历史

http协议从1989年发明以来，就迅速得得到推广，在1994年由网景公司引入安全的http协议，即http over ssl，也就是https协议。ssl从1.0快速迭代到1996年的SSL 3.0，随后在3.0版本基础上引入了TLS加密协议。截止到2015年，SSL3.0已经停止维护。现在网络上已经基本上是TLS协议。目前TLS发展到了TLS1.3版本。

### https的使用率

谷歌通过chrome浏览器进行跟踪统计发现互联网的绝大部分流量都是https协议。这要归功于整个行业的安全意识的普及和几个互联网巨头在浏览器上的强制推广。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/1/472cc224-583d-46b6-beff-b5484be8320b.jpeg?raw=true)

而对于内网的流量传输，没有一个公开的统计，以运营商的一些试点来看，https协议也达到了80%以上的覆盖率。

二、现状问题
------

1、通道安全与流量分析的矛盾

https协议使得通道传输更加安全，中间人攻击现像大大减少，再也不会在浏览网页的时候出现莫名其妙的广告。不过中间人无法窃听了，自己出于业务需要也无法监听，因为需要对https传输内容进行统计分析。这种矛盾产生了很多https解密解决方案。

**部署环境的要求**

我们先看一个主流的B/S架构

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/1/efaa5b27-fa0a-4bb8-8d21-0fe19bb35e90.jpeg?raw=true)

客户端与web服务端的传输，中间会经过交换机，网关，代理服务器等。每个过程都是串联部署，自然地就对数据进行过滤，对于https流量，没有获取到证书的时候，中间的网关无法直接看到http原始数据。那么我们将基于这样的架构，来讨论从不同的时机地点截取并解密加密协议流量。

三、解决方案
------

### 1、中间代理

代理网关，作为串联的部署的一部分，桥接其客户端和服务端的中间件。如果能够得到服务端的私钥，那么就可以作为合法的中间人。对于客户端来说，他是服务端，与客户端建立https连接；对于服务端来说，他是客户端，与服务端建立https连接。主流的像nginx服务就经常充当这样的角色。中间人非常方便的将解密后的数据传输或者存储到第三服务器，并用于进一步分析。部署架构如下：

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/1/cadf0f8d-4616-4a2a-a129-9575869563de.jpeg?raw=true)

优势：

对客户端和服务端无感知，无需改造程序，能够精准的得到传输的所有数据。

缺点：

需要做网络改造，中间植入代理网关，增加访问链，增加风险。由于做日志提取分析的客户往往并不是业务部门，而是安全部门，无法对网络拓扑进行改造，所以这个方案接受度不是很高。

### 2、旁路解密

另一种接受度高的方案是：通过旁路镜像流量的方式，采集到流量之后进行解密，需要导入服务端的私钥证书。这种方案无需改造双端程序，无需改造网络拓扑，零侵入，零影响。部署架构如下：

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/1/22fd94df-830b-4ce1-b154-a027ac34ec60.jpeg?raw=true)

这种方式的缺点是：旁路流量有可能会丢包，导致解密不全。另一方面，业务上有可能使用了前向安全的加密协议，这种协议通过旁路即使有私钥证书也是无法解密的。因为非常多的客户对这个不理解，我们稍微展开讨论。

在https的加密协议其实包含了两部分：握手阶段的非对称密钥交换协议和会话对称加密阶段。之所以要两种加密协议，是因为非对称加密是通道相对安全的，但是性能较低。而对称加密性能高，但是双端的密钥在传输通道是明文的。所以实现上就先利用非对称加密协议协商出一个随机的密钥，用于对称加密的密钥。非对称密钥交换协议，主流的有RSA和ECDHE两种协议。协议过程如下：

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/1/7d3e3987-2ad9-4e54-adcb-b4e41d3036c2.jpeg?raw=true)

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/1/142abcae-96e7-4e31-945e-9708fe2e0798.jpeg?raw=true)

**（图从阿里云公开文章引用，版本归阿里云所有）**

从上面可以看到，RSA的密钥协商过程在传输通道里其实传输了**Client Random、Server Random****，服务器公钥，加密的预主密钥**。通过双端随机数、服务器私钥、和加密的预主密钥可以解密出预主密钥。预主密钥可以转换成主密钥和会话密钥。而这些数据都可以旁路抓包获取到。

相比之下，ECDHE协议在通道里传输了**Client Random、Server Random 、使用的椭圆曲线、椭圆曲线基点 G、服务端椭圆曲线的公钥**。市面上所有旁路解密ssl的厂商都会强调，无法解析前向安全的加密方式。而随着TLS1.3的协议普及，越来越多的应用开始升级应用，旁路解密的方法也越来越局限。这里有一篇文章详细描述了会话过程：http://www.likecs.com/default/index/show?id=124371

### 3、 应用端解密

应用端，有时候也叫终端。目前市面上主流的终端DLP，大多可以解密SSL。一些流量抓包软件，比如wireshark，fiddler等在终端旁路抓包的情况下，可以通过浏览器（chrome）直接导出会话密钥的方式来解密。终端解密的方案需要在用户终端安装插件，而且部署和插件管理相对比较复杂。

chrome支持在环境变量设置一个值：SSLKEYLOGFILE，为其指定一个文件路径，chrome浏览器会将密钥保存这个文件里，在wireshark软件配置ssl参数，让他读取该值，即可完成https的协议解析。

### 4、生产端解密

4.1、应用程序hook机制

很多应用程序是java，php等程序开发，支持通过启动程序的时候在进程内部植入agent插件的形式，进行抓取请求和返回的http数据，这种方式处于应用层解密之后的处理阶段，得到的信息与旁路抓包信息一致，但是利用了程序自身可以被hook的特性来间接达到效果。这种形式比打印日志，然后读取本地日志来的更方便和高效。缺点就是需要植入插件，干扰程序，共用进程会降低运行的稳定性。适用于在一些不愿意改造程序，又希望能截获日志的场景。

在12年的时候，Gartner引入了RASP技术，这种技术通过将Hook程序注入到应用程序中，与业务融为一体。对于java程序，本质是利用java agent的机制来实现，自然就可以用来将http信息传输到第三方应用进一步分析利用。百度开源了一个框架叫OpenRasp，https://rasp.baidu.com/ ，即是一个很好的实现。

4.2、系统组件hook

由于大部分应用程序的ssl的实现都是openssl这类组件库，所以基于C语言的hook函数机制，可以实现指定进程的组件函数hook。加密阶段，在调用Encrypt加密函数的时候，传入的参数就是明文数据。在hook函数里，直接把源数据进行保存或者外发，然后调用原来的加密函数。解密过程同理。这种方式的优势是，可以解密前向加密问题，无需在应用程序植入agent。而且由于逻辑简单，不会对产品的业务产生影响。但是也是属于半串联的部署模式，而且毕竟还是在业务主机上实施，会对产品的稳定性产生一定影响。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/1/0cdceaea-fb1d-4dae-bdfb-1b22827c4c0e.jpeg?raw=true)

谷歌封装了一个库叫ssl_logger : [https://github.com/google/ssl_logger](https://github.com/google/ssl_logger)，这个库底层依赖于轻量级Hook框架：Frida，可以在应用层进程解密。

四、总结
----

数据安全传输和数据安全监测，并不总是并行不悖的，有一种隐含的矛盾在里面，所以怎么能更好的弥合这种矛盾，各取所需，是很关键的。在以上所列举的几种解决方案里，分别适用于不同场景，不同客户类型，包括一线运维人员、IT部门领导、安全管理员、业务负责人，都会有偏好，需要根据实际情况来考虑https解密方案。