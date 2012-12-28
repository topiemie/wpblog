"=========== Meta ============
"StrID : 243
"Title : SSH Agent Forwarding原理
"Slug  : ssh-agent-forwarding-guide
"Cats  : Uncategorized
"Tags  : ssh, ssh-agent
"=============================
"EditType   : post
"EditFormat : Markdown
"BlogAddr   : http://blog.pkufranky.com/
"========== Content ==========
$TOC$

ssh-agent的manual写得倒是挺详细，可看了好几次都没怎么搞明白。08年在网上找到了非常好的一篇文章，[An Illustrated Guide to SSH Agent Forwarding][agent-forwarding-guide] (后文简称agent guide), 将ssh的各种认证方法讲得非常之详细。 文章从密码认证，公钥认证，使用agent以及agent forward的公钥认证几个方面，逐步的将整个过程剖析得非常全面。 看完之后总算是入了门，为此写了一篇简短的[博文][ssh-agent]。

本周为了做ssh agent相关的培训，多方查看资料，包括ssh/sshd的manual, 相关RFC, wikipedia。 终于算是把密码学和ssh相关东西理解得更深入了。然后重新将agent guide看了看，发现了一些问题。agent guide是2006年写的，而06年SSH-2刚刚出来，因此文章是基于SSH-1的。虽然ssh agent的基本原理还是对的，但有的地方(主要是认证部分)已经不正确了。

所以本文针对SSH-2，将不准确的地方重新梳理一下，并添加了一些SSH基石(密码学)相关的内容。
<!--more-->

# 密码学

先从密码学说起。 现代以前，密码学(cryptography)主要专指加密(encryption)解密(decryption)。一组配对的加密和解密算法称为cipher. 加解密的具体运作由两部分决定：一个是算法(algorithm)，另一个是钥匙(key).

现代密码学主要分为对称钥匙密码学和公钥密码学(非对称钥匙密码学)。对称钥匙密码学指的是加密方和解密方都拥有相同的密钥。对称钥匙加密算法包括3DES, AES, Blowfish等。 对称钥匙密码学依赖于钥匙的保密性，其尴尬的难题是：当安全的通道不存在于双方时，如何安全传送这一双方共享的密钥？如果有通道可以安全地建立钥匙，何不使用现有的通道。这个“鸡生蛋、蛋生鸡”的矛盾是长年以来密码学无法在真实世界应用的阻碍。直到1976年，公钥密码学的诞生，安全通道的问题才得以很好的解决。这一点下面讲SSL/TLS的时候会提到。

公钥密码学，则使用一对公钥和私钥，通过公钥加密，私钥解密。公钥和私钥是相关的，但很难从一个推导出另外一个。公钥密码学不存在安全传送密钥的问题，因为公钥可以对外公开，明文传送。

公钥密码学包括公钥加密算法和数字签名算法。[RSA][]是最常见的公钥加密算法。RSA算法由3步构成: 公钥私钥的生成，加密和解密。公钥和私钥中的任何一个可用作加密，另一个则用作解密。

RSA也可用作数字签名，甲方将消息的散列值使用私钥加密，作为签名附在消息后面，乙方收到消息后使用公钥将签名解密，然后和消息计算的散列值进行对比。假如两者相符的话，那么乙方就可以知道发信人持有甲的私钥，以及这个消息在传播路径上没有被篡改过。

[DSA][]是常用的数字签名算法，但不能用作加密解密。

如何利用密码学来安全传输数据呢？目前最常用的安全数据传输协议(应用层协议)是[SSL/TLS][TLS]. SSL (Secure Socket Layer)协议分为1.0, 2.0(1995), [3.0(1996)][SSL3.0]三个版本。TLS (Transport Layer Security)则是SSL的后继协议，分为1.0, 1.1, [1.2(2008)][TLS1.2]三个版本。TLS在SSL3.0的基础上改动并不大，但和SSL3.0不能互操作(interoperate)。TLS中包含将连接降级到SSL3.0的方法，因此也写作SSL/TLS.


TLS一般使用基于非对称密码学的[Diffie-Hellman key exchange][]来安全传送共享密钥，然后使用对称钥匙加密算法以及这一共享密钥对传送的数据进行加密。很多应用层的协议都可通过SSL/TLS来安全传输数据，比如最常见的HTTPS, 邮件传输服务协议(SMTP)等。

> Secure Sockets Layer (SSL) uses Diffie-Hellman key exchange if the client does not have a public-private key pair and a published certificate in the Public Key Infrastructure, and Public Key Cryptography if the user does have both the keys and the credential.

# ssh认证和agent forwarding

终于到正题了，下面开始讲SSH.

那[SSH][] (Secure Shell)和SSL/TLS是什么关系呢？SSH也是一个网络协议，用来进行安全数据交流，远程shell服务和命令执行等。SSH由传输，认证和连接等协议组成。SSH的传输协议类似SSL/TLS (Diffie-Hellman key exchange以及对称钥匙加密)。

本文的重点是SSH的认证部分。client和server通过key exchange获得共享密钥(shared session key)后，所有之后的传输数据都进行了加密。然后进入认证部分，认证成功后，则双向连接通道建立，通常是login shell.

来自securecrt官网[关于shared session key的描述](http://www.vandyke.com/solutions/ssh_overview/ssh_overview_encryption.html)

> Session keys are the "shared keys" described above and are randomly generated by both the client and the server during establishment of a connection. Both the client and host use the same session key to encrypt and decrypt data although a different key is used for the send and receive channels. Session keys are generated after host authentication is successfully performed but before user authentication so that usernames and passwords can be sent encrypted. These keys may be replaced at regular intervals (e.g., every one to two hours) during the session and are destroyed at its conclusion.


[SSH认证][RFC4252]有多种方法，本文着重讲最常见了两种：密码认证和公钥认证。

## 1. 密码认证

密码认证最简单：

1. ssh client向目标机器发起tcp连接(一般22端口)并发送username (username是SSH协议的一部分)
1. 目标机器ssh daemon回应需要密码
1. ssh client提示用户输入密码，然后将密码发送到服务器端
1. ssh daemon如果密码匹配成功, 则认证通过，

基于密码认证的缺点是

- 容易被brute-force password guessing
- 不适合于管理多台机器
若每台机器使用相同的密码，如果密码泄露，所有机器都被攻破。若使用不同密码，则密码太多很难记住，因此也不可能使用很强的密码。

## 2. 公钥认证

公钥认证详细协议见[RFC4252的publickey部分](http://tools.ietf.org/html/rfc4252#section-7)

公钥认证需要先在本地机器生成公钥私钥对，然后将公钥放到目标机器的$HOME/.ssh/authorized_keys中。具体过程如下

1. ssh client向目标机器发起tcp连接(一般22端口)
1. ssh client提示用户输入passphrase以解密私钥
1. ssh client发送私钥签名的包含username和公钥等信息的message. 
1. 目标机器ssh daemon通过检查消息中指定用户的`$HOME/.ssh/authorized_keys`，确定公钥是否可用作认证并验证签名的合法性, 如果两者都ok, 则通过认证

如果公钥认证失败，ssh还会尝试其他认证策略，比如密码认证。多个认证策略的尝试顺序和服务器端没关系，由客户端的配置来决定。

需要说明的是，即使把本机的公钥(如`.ssh/id_rsa.pub`)删除掉，认证仍然可以成功。那第三步中提到的公钥从哪里来的呢？实际上，上面(如第二步)提到的私钥(如`.ssh/id_rsa`)是广义的，既包含了私钥，也包含了公钥，也有可能还包含了其他信息(比如证书)。比如通过`ssh-keygen -y ~/.ssh/id_rsa`就可以看到`id_rsa`里面的公钥。

用作认证的私钥最好通过passphrase进行加密，否则会有很大安全隐患，只要私钥泄露，别人就能访问你能访问的所有远程机器。

公钥认证由于需要配置公钥私钥，初始配置稍微麻烦一些，但好处是所有机器只需配置一组公私钥对就可以了。由于只有一个私钥，不必设置多个密码，因此可以为其设置比较强的密码。并且仅当私钥和密码一同丢失时才有风险，但这个概率非常小。

不过仍然烦人的是，每次登陆都得输入passphrase。

## 3. 使用ssh agent的公钥认证

为解决每次登陆远程机器都需要输入passphrase的问题，ssh-agent被引入了。ssh-agent启动后，可通过ssh-add将私钥加入agent. ssh-add会提示用户输入passphrase以解密私钥，然后将解密后的私钥纳入agent管理。agent可同时管理多个私钥。

连接服务器的步骤如下:

1. ssh client向目标机器发起tcp连接(一般22端口)
1. ssh client向本地的agent请求, 得到私钥签名的包含username和公钥等信息的message
1. ssh client向目标机器发送此message和签名
1. 目标机器ssh daemon通过检查消息中指定用户的`$HOME/.ssh/authorized_keys`，确定公钥是否可用作认证并验证签名的合法性, 如果两者都ok, 则通过认证

如果ssh-agent中有多个私钥, 会依次尝试，直到认证通过或遍历所有私钥. 

在整个过程中，私钥只存在于agent的内部(内存中), ssh client并没有获取到私钥。

使用ssh-agent后，只需在将key纳入agent管理时输入passphrase，之后的ssh相关操作就不必输入passphrase了。但如果从本机A登陆机器B后，又想从B登陆C (或从B传输文件到C)，仍然需要输入passphrase (如果B上也配置了用户的私钥)或password。还是比较麻烦。

幸好，ssh agent forwarding解决了这一问题。

## 4. 使用ssh agent forwarding的公钥认证

1.  假设用户已经从homepc连接到了第一台机器server。homepc的agent中已保存了用户的私钥
1.  server: 用户从server向server2发起ssh连接请求
1.  server: ssh client向本地(server)的agent请求, 得到私钥签名的包含username和公钥等信息的message。

	注意server上其实ssh-agent压根就没有启动，ssh client只是检查`$SSH_AUTH_SOCK`这个环境变量是否存在，如果存在，则和这个变量指定的domain socket进行通信。而这个domain socket其实是由server上的sshd创建的。所以ssh client其实是和sshd在通信。

	而server的sshd并没有私钥信息，所以sshd做的事情其实是转发该请求到homepc的ssh client，再由该client将请求转发给本地(homepc)的agent。该agent将需要的消息和签名准备完毕后，再将此数据按原路返回到server的ssh client. 路径如下所示

		agent_homepc --($SSH_AUTH_SOCK)-- ssh_homepc --(tcp)-- \
		sshd_server --($SSH_AUTH_SOCK)-- ssh_server --(tcp)-- \
		sshd_server2

	这下明白为什么叫agent forwarding(转发)了吧，就是所有中间节点的sshd和ssh都充当了数据转发的角色，一直将私钥操作的request转发到了本机的agent，然后再将agent的response原路返回。

1.  server: ssh client向目标机器server2发送此message和签名
1.  server2: ssh daemon通过检查消息中指定用户的`$HOME/.ssh/authorized_keys`，确定公钥是否可用作认证并验证签名的合法性, 如果两者都ok, 则通过认证

上面只是示例，从server2，还可以类似的无密码登陆到server3。事实上，通过ssh agent forwarding, 能实现任意级别的无密码登陆。并且私钥只保存在本地的机器上，保证了私钥的安全。

agent forwarding功能是默认关闭的，为实现任意级别无密码登陆，在ssh到其他机器时，一定要记得添加`-A`参数， 以打开agent forwarding (在目标机器上会生成$SSH_AUTH_SOCK环境变量)。 比如 `ssh -A server2`。

agent forwarding打开之后，也会有安全的风险。如果用户A通过ssh连接server并打开了agent forwarding，因为server上的root用户也有权限访问与agent通信的套接字，只要root用户将`$SSH_AUTH_SOCK`指向用户A对应的套接字地址，就可以以A的身份访问其它A可以访问的机器。因此请确保agent forwarding仅在可信任的服务器上打开。

本文主要从基本原理角度对ssh认证和agent相关问题进行了分析。[下文][using-ssh-agent-forwarding]会讲讲最佳实践。

[RSA]: http://en.wikipedia.org/wiki/RSA_(algorithm)
[DSA]: http://en.wikipedia.org/wiki/Digital_Signature_Algorithm
[TLS]: http://en.wikipedia.org/wiki/Transport_Layer_Security
[symmetric-key]: http://en.wikipedia.org/wiki/Symmetric-key_algorithm
[public-key]: http://en.wikipedia.org/wiki/Public-key_cryptography
[SSL3.0]: http://tools.ietf.org/html/rfc6101
[TLS1.2]: http://tools.ietf.org/html/rfc5246
[Diffie-Hellman key exchange]: http://en.wikipedia.org/wiki/Diffie%E2%80%93Hellman_key_exchange
[SSH]: http://en.wikipedia.org/wiki/Secure_Shell
[RFC4252]: http://tools.ietf.org/html/rfc4252
[agent-forwarding-guide]: http://www.unixwiz.net/techtips/ssh-agent-forwarding.html
[ssh-agent]: http://pkufranky.blogspot.hk/2008/04/ssh-agent.html
[using-ssh-agent-forwarding]: /2012/08/using-ssh-agent-forwarding/
