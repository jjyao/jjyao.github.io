---
layout: post
title: "Does Your App Have a Rock Solid SSL Connection"
date: 2013-07-31 20:32
comments: true
categories:
---
SSL(Secure Sockets Layer)是目前广泛用来加密网络通信的加密协议。它的一个著名应用就是HTTPS，也就是使用SSL协议对HTTP进行加密从而使得整个通信更加安全。对于一个移动应用（其他类型的客户端也同理），为了保证其和服务器的通信安全，开发者就会使用SSL来加密客户端和服务器之间的通信。这样理应是安全的，但由于很多开发者使用方式不对，导致客户端和服务器之间的SSL通信会受到中间人攻击(Man-in-the-middle attack)，从而使得安全性荡然无存。本文将首先介绍下SSL和中间人攻击的原理，然后会提供一个中间人攻击的实例，最后本文介绍该如何防止中间人攻击。

<!-- more -->

##SSL原理
SSL加密本质上就是用一个只有客户端和服务器知道的对称密钥加密网络上的数据包以达到保密的目的。在这里最主要的问题是如何产生**只有**客户端和服务器知道的对称密钥，SSL协议解决这个问题的过程被称为SSL的握手(handshake)。下面就简要描述下客户端和服务器建立了socket连接后SSL握手的过程（这是个简化版的描述，详细内容可见本文给出的参考资料）：  

1. 客户端向服务器发送一个ClientHello消息（明文），包括了客户端支持的一系列加密算法、压缩算法和一个随机数(ClientRandom)。  
2. 服务器接收到ClientHello后就可以知道客户端支持哪些加密算法和压缩算法，然后服务器就可以选择一个客户端和服务器都支持的加密算法和压缩算法来作为接下来SSL连接中要使用的算法。选择好算法后服务器向客户端发送一个ServerHello消息（明文），包括了选择好的加密算法、压缩算法和一个随机数(ServerRandom)。  
3. 发送完ServerHello消息后，服务器紧接着又发送了一个Certificate消息（明文），包括了服务器的证书。这个证书是由CA颁发用来验证服务器身份的，里面包含了服务器的公钥等信息。  
4. 发送完Certificate消息后，服务器又发送了一个ServerHelloDone消息（明文），这是一个空的消息，表明服务器已经发送了这阶段要发送的全部信息，等待客户端的反馈。  
5. 客户端验证服务器证书的有效性，如果确认有效那么客户端会向服务器发送一个ClientKeyExchange消息（密文），这个消息由服务器证书中提供的公钥加密，包括了一个PreMasterSecret。PreMasterSecret是一个48字节的值，由两个版本号字节以及46个随机产生的字节组成，可以将其看做是一个随机数。 
6. 服务器收到ClientKeyExchange消息后用自己的私钥解密就可以得到PreMasterSecret。至此，客户端和服务器都拥有了三个随机数，分别是：ClientRandom, ServerRandom和PreMasterSecret。然后客户端和服务器都运行同一个密钥导出函数，将上述三个随机数作为input，产生的output中就包含了接下来用来加密应用数据的对称密钥。   
7. 发送完ClientKeyExchange消息后，客户端又发送了ChangeCipherSpec消息和Finished消息来表明客户端已结束握手过程。  
8. 最后服务器也发送了ChangeCipherSpec消息和Finished消息来表明服务器已结束握手过程。  
9. 到此为止整个SSL握手过程就结束了，接下来客户端和服务器就开始用握手阶段得到的对称密钥来加密应用数据。  

从上面描述的握手过程可以得知PreMasterSecret是一个**只有**客户端和服务器知道的值（因为它是客户端生成的，所以客户端知道。同时它是用服务器的公钥加密后传给服务器的，所以只有服务器手上的私钥能够解密）。而PreMasterSecret是产生对称密钥的一个input，由于input只有客户端和服务器知道，因此产生的output也就是对称密钥就只有客户端和服务器知道了，所以使用SSL来加密通信数据是安全的。有人可能会问那么ClientRandom和ServerRandom是用来干什么的呢（剧透：防止replay攻击）？使用SSL加密后的数据是无法被第三者解密了，但是又如何保证数据的完整性呢（剧透：使用MAC）？SSL是不是还能防止其他各类攻击（剧透：还能防止截断攻击等）？SSL不是用来加密数据的么，为什么还涉及数据压缩（剧透：压缩得在加密之前）？上述的握手过程只是一个简化版本，那么完整的握手过程又是怎样的呢（剧透：完整的握手可能还涉及到会话恢复和客户端认证等）？上述问题以及可能产生的其他关于SSL的疑问都可以在《SSL and TLS - Designing and Building Secure Systems》中找到答案，这本书写的非常好，值得一读（中文翻译的也还行）。

##中间人攻击原理
中间人攻击的本质就是客户端以为它在和服务器通信，实际上它是在和中间人通信。服务器以为它在和客户端通信，实际上它也在和中间人通信。对于SSL来说，客户端和服务器都以为它们和对方建立了一条SSL连接，而实际上它们分别和中间人建立了一条SSL连接。这样的后果就是中间人能够得到客户端和服务器之间传输数据的明文，还能任意更改数据。下图是中间人攻击的示意图：

{% img /images/post/is-ssl-security-of-your-app-rock-solid/MITM.png %}

在上图中Mallory就是中间人，在Alice（客户端）和Bob（服务器）建立SSL连接的过程中，Mallory截获Alice发出的ClientHello等消息，然后假装自己是Bob回应ServerHello等信息，这样Alice和Mallory之间就建立了一条SSL连接。同时Mallory假装自己是Alice，和Bob之间也建立了一条SSL连接。这样Alice发出的数据Mallory可以解密并修改，然后Mallory可以将数据通过它和Bob之间的SSL连接再传给Bob。而Bob的返回数据Mallory也可以解密并修改，然后再传给Alice。

从上面的描述可以看到中间人攻击的关键就是Alice把Mallory当成了Bob，而这实际上可以通过SSL握手过程中的验证服务器证书来避免的。只要Alice仔细检查收到的证书确实是Bob的，那中间人攻击就失效了。因为Mallory给Alice的一定是Mallory的证书，只有这样Mallory才有私钥来解密PreMasterSecret。假设Mallory给Alice的是Bob的证书，那么Mallory是没有Bob的私钥去解密PreMasterSecret的，也就说其不能得到对称密钥，自然也就无法解密Alice加密后的数据。

##中间人攻击实例
很多App（客户端）的开发者没有正确地检查建立SSL连接时收到的服务器证书，从而使得中间人攻击成为了可能。我就对自己手机上装的一个App成功实施了中间人攻击，获取了它和服务器传输的内容，下面就简要介绍下攻击的过程：

1. 首先是在Mac上安装Charles，这是一个sniffer工具，支持中间人攻击。 
2. 然后配置iPhone和Mac，使得iPhone把Mac作为代理连接网络，这样Charles就有机会截获iPhone上发出的数据包了。  
3. 接着在iPhone上装Charles的根CA证书。Charles作为中间人发给客户端的是自己CA签署的证书，而这个CA不是iPhone默认认为的可信任CA，所以要手动把Clarese的CA加入可信任CA列表。
4. 打开手机上的App，然后在Charles中就可以发现App发出的数据了（尽管App使用的是SSL连接）。

更详细的做法请参见[这里](https://www.cocoanetics.com/2010/12/how-to-spy-on-the-web-traffic-of-any-app/)。下面就给出几张攻击的截图：

{% img /images/post/is-ssl-security-of-your-app-rock-solid/example1.png %}

{% img /images/post/is-ssl-security-of-your-app-rock-solid/example2.png %}

{% img /images/post/is-ssl-security-of-your-app-rock-solid/example3.png %}

从截图中可以看出我已经获得了这个应用和服务器传输的所有数据。

##防止中间人攻击
为了防止中间人攻击，客户端需要检查SSL握手时获得的证书确实来自其想要连接的服务器，使用的技术叫做Certificate and Public Key Pinning。最简单的做法就是在客户端存放服务器的证书，然后每次建立SSL连接的时候都将远程获得的证书和本地的相比较，只有两者完全一样才认为没有受到中间人攻击，否则就应该拒绝建立连接。这种做法虽然实现简单，但问题是证书的内容很容易改变（比如证书过期后就需要重新获得一个全新的证书），因此每当证书变化时旧版应用就无法连接服务器了，需要让用户升级应用，对用户来说这显然不够好。另一种较好的方法是只比较证书中subjectPublicKeyInfo部分而不是整个证书。相对于整个证书而言，这个部分变动概率很小并且其包含了要验证的最主要东西也就是服务器的公钥，因为只要公钥对了，那么加密的数据就只有拥有对应私钥的服务器才能解密了。对于pinning的更多资料请参见[这里](https://www.owasp.org/index.php/Certificate_and_Public_Key_Pinning)。
