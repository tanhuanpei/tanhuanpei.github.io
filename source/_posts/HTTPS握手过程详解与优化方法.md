---
title: HTTPS握手过程详解与优化方法
date: 2019-01-29 16:14:23
tags: HTTPS 优化
---

#### HTTPS

HTTPS协议可以简单的认为是HTTP+TLS/SSL。`SSL（Secure Socket Layer）是安全套接层，TLS（Transport Layer Security）是传输层安全协议`，建立在SSL3.0协议规范，是SSL3.0的后续版本。当前最新使用的是TLS1.2协议。1.3版本还在草案阶段。

![image](/HTTPS握手过程详解与优化方法/1548387314931.png)



HTTPS 在 HTTP 下面提供了一个传输级的密码安全层——可以使用 SSL，也可以使用其后继者—— `传输层安全(Transport Layer Security，TLS)`。由于 SSL 和 TLS 非常类似，所以我们不太严格地用术语 SSL 来表示 SSL 和 TLS。

![image](/HTTPS握手过程详解与优化方法/930623111-59ed85819d22b_articlex.png)

#### TLS握手过程

在传输应用数据之前，客户端与服务端经过协商秘钥、加密算法等信息，服务端还要把自己的证书发送给客户端以表明身份，这些环节构成了TLS握手过程，如下两图所示：

![img](https://upload-images.jianshu.io/upload_images/2111324-19f47f0a6829c6f1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/409/format/webp)



![](/HTTPS握手过程详解与优化方法/tsl-ssl.png)

##### 第一步

ClientHello【client -> server】

1. TLS版本
2. Client随机数**Random1**
3. session-id
4. 加密算法套装列表
5. 压缩算法
6. 扩展字段，比如密码交换算法，请求主机名字等

![1548387743050](/HTTPS握手过程详解与优化方法/1548387743050.png)

##### 第二步

ServerHello【server -> client】

1. TLS版本

2. Server随机数**Random2**

3. 确定的加密套件，压缩算法

4. 扩展字段

   ![1548388040046](/HTTPS握手过程详解与优化方法/1548388040046.png)



##### 第三步

Certificate, Server Key Exchange, Server Hello Done【server -> client】

- Certificate，CA证书
- Server Key Exchange，密钥交换算法和相关数据
- Server Hello Done，事件消息

![1548397164402](/HTTPS握手过程详解与优化方法/1548397164402.png)



##### 第四步

Client Key Exchange, Change Cipher Spec, **Encrypted Handshake Message**【client -> server】

- Client Key Exchange，交换密钥参数

  **这里客户端会再生成一个随机数Random3。然后使用服务端传来的公钥进行加密得到密文PreMaster Key。服务端收到这个值后，使用私钥进行解密，得到Random3。这样客户端和服务端就都拥有了Random1、Random2和Random3。这样两边的秘钥就协商好了。后面数据传输就可以用协商好的秘钥进行加密和解密。**

- Change Cipher Spec，编码改变通知。

  这一步是客户端通知服务端后面再发送的消息都会使用前面协商出来的秘钥加密了，是一条事件消息。

- Encrypted Handshake Message

  这一步对应的是 Client Finish 消息，客户端将前面的握手消息生成摘要再用协商好的秘钥加密，这是客户端发出的第一条加密消息。服务端接收后会用秘钥解密，能解出来说明前面协商出来的秘钥是一致的。

  **会话秘钥 = DES(random1+random2+rsa(ramdom3))**

![1548398905865](/HTTPS握手过程详解与优化方法/1548398905865.png)



##### 第五步

New Session Ticket, Change Cipher Spec, **Encrypted Handshake Message**【server -> client】

- New Session Ticket

  包含了一个加密通信所需要的信息，这些数据采用一个只有服务端知道的密钥进行加密。目标是消除服务器需要维护每个客户端的会话状态缓存的要求。

- Change Cipher Spec，编码改变通知

- Encrypted Handshake Message

  这一步对应的是Server Finish消息，服务端也会根据握手过程的消息生成摘要再用秘钥加密，这个服务端发出的第一条加密消息。客户端接收后会用秘钥解密，能解出来说明协商的秘钥是一直的。

![1548399295697](/HTTPS握手过程详解与优化方法/1548399295697.png)

至此，双方SSL/TLS握手结束。



#### HTTPS优化

- HTTP协议优化

  - HTTP2优势

    升级HTTP2，HTTP2主要有以下特性：

    1. 二进制分帧，数据使用二进制传输，相比于文本传输，更利于解析和优化。
    2. 多路复用，同一域名下的请求，共用同一条链路进行传输，有效节省消耗。
    3. 头部优化，将头部字段缓存为索引，客户端与服务端维护索引表，通信过程中尽可能采用索引进行通信，收到索引后查询索引表，才能解析出真正的头部信息。

  - 兼容性问题

    需要注意的是，HTTP2在TLS层的协议协商使用的是NPN（Next Protocol Negotation）协议或者ALPN（Application Layer Protocol Negotation）协议。ALPN和NPN的主要区别在于：谁来决定通信协议。在ALPN的描述中，是让客户端先发送一个协议优先级列表给服务器，由服务器最终选择一个合适的。而NPN则正好相反，客户端有着最终的决定权。**客户端和服务端都支持NPN或ALPN协商，是用上HTTP/2的大前提。**

    OkHttp是Square公司开源的客户端网络请求库，天然支持HTTP2，但在最新版本上，已经移除了对NPN协议的支持，转而只支持ALPN协议。ALPN协议只支持Android 5.0以上，如果在5.0以下支持HTTP2，必须使用NPN。

    服务器理论上可以对NPN和ALPN同时支持，但部分服务器配置可能只支持NPN，并且短时间内不会支持ALPN，所以要用上HTTP2，必须使用NPN。

    也就是说，根据服务器支持情况和手机系统版本决定是否开启HTTP2/。

- TLS握手优化

  1. Session Resumption，会话复用

     - Session Id，服务端开启
  
   - Session ticket，服务端支持，客户端开启
  
     在Android API文档中``SSLCertificateSocketFactory``提供了开启Session ticket的方法
  
     ```java
       /**
          * Enables <a href="http://tools.ietf.org/html/rfc5077#section-3.2">session ticket</a>
            * support on the given socket.
          *
            * @param socket a socket created by this factory
          * @param useSessionTickets {@code true} to enable session ticket support on this socket.
            * @throws IllegalArgumentException if the socket was not created by this factory.
          */
           public void setUseSessionTickets(Socket socket, boolean useSessionTickets){
             castToOpenSSLSocket(socket).setUseSessionTickets(useSessionTickets);
           }
     ```
  
       最终实现在[OpenSSLSocketImpl](https://android.googlesource.com/platform/external/conscrypt/+/f087968/src/main/java/org/conscrypt/OpenSSLSocketImpl.java?autodive=0%2F%2F%2F%2F)中，如下代码：

       ```java
     /**
            * This method enables session ticket support.
          *
            * @param useSessionTickets True to enable session tickets
          */
           public void setUseSessionTickets(boolean useSessionTickets) {
             this.useSessionTickets = useSessionTickets;
           }
     ```
  
       需要使用的时候可以通过自定义SSLSocketFactory和反射来启动session ticket。

       ```java
     public class SSLExtensionSocketFactory extends SSLSocketFactory {
           private String TAG = "TlsSessionTicket";
         private SSLSocketFactory mDelegate;
       
         public SSLExtensionSocketFactory(Context context) {
               try {
                 SSLContext sslContext = SSLContext.getDefault();
                   SSLSessionCache sslSessionCache;
                 try {
                       sslSessionCache =
                               new SSLSessionCache(new File(Environment.getExternalStorageDirectory(), "sslCache"));
                   } catch (IOException e) {
                       DebugLogger.e(TAG, e.getMessage());
                       sslSessionCache = new SSLSessionCache(context);
                   }
                   install(sslSessionCache,sslContext);
                   mDelegate = sslContext.getSocketFactory();
               } catch (Exception e) {
                   DebugLogger.e(TAG,e.getMessage());
                   mDelegate = (SSLSocketFactory) SSLSocketFactory.getDefault();
               }
           }
       
           private Socket makeSSLSocketSessionTicketSupport(Socket socket) {
               if(socket instanceof SSLSocket) {
                   setUseSessionTickets(socket);
               }
               return socket;
           }
           @Override
           public String[] getDefaultCipherSuites() {
               return mDelegate.getDefaultCipherSuites();
           }
       
           @Override
           public String[] getSupportedCipherSuites() {
               return mDelegate.getSupportedCipherSuites();
           }
       
           @Override
           public Socket createSocket(Socket s, String host, int port, boolean autoClose) throws IOException {
               return makeSSLSocketSessionTicketSupport(mDelegate.createSocket(s, host, port, autoClose));
           }
       
           @Override
           public Socket createSocket(String host, int port) throws IOException {
               return makeSSLSocketSessionTicketSupport(mDelegate.createSocket(host, port));
           }
       
           @Override
           public Socket createSocket(String host, int port, InetAddress localHost, int localPort) throws IOException, UnknownHostException {
               return makeSSLSocketSessionTicketSupport(mDelegate.createSocket(host, port, localHost, localPort));
           }
       
           @Override
           public Socket createSocket(InetAddress host, int port) throws IOException {
               return makeSSLSocketSessionTicketSupport(mDelegate.createSocket(host, port));
           }
       
           @Override
           public Socket createSocket(InetAddress address, int port, InetAddress localAddress, int localPort) throws IOException {
               return makeSSLSocketSessionTicketSupport(mDelegate.createSocket(address, port, localAddress, localPort));
           }
       
           private void setUseSessionTickets(Socket socket){
               Class c = socket.getClass();
               try {
                   Method m = c.getMethod("setUseSessionTickets",boolean.class);
                   m.invoke(socket,true);
               } catch (NoSuchMethodException e) {
                   e.printStackTrace();
               } catch (IllegalAccessException e) {
                   e.printStackTrace();
               } catch (IllegalArgumentException e) {
                   e.printStackTrace();
               } catch (InvocationTargetException e) {
                   e.printStackTrace();
               }
           }
       
           private void install(SSLSessionCache sslSessionCache,SSLContext sslContext){
               Class c = sslSessionCache.getClass();
               try {
                   Method method = c.getMethod("install", new Class<?>[] {SSLSessionCache.class, SSLContext.class});
                   method.invoke(sslSessionCache,sslSessionCache,sslContext);
               } catch (NoSuchMethodException e) {
                   e.printStackTrace();
               } catch (InvocationTargetException e) {
                   e.printStackTrace();
               } catch (IllegalAccessException e) {
                   e.printStackTrace();
               }
           }
       }
       ```
  
       
  
       Okhttp中的`Realconnection`已默认开启session ticket。
  
       ```java
       private void connectTls(ConnectionSpecSelector connectionSpecSelector) throws IOException {
             ······
             if (connectionSpec.supportsTlsExtensions()) {
               Platform.get().configureTlsExtensions(
                   sslSocket, address.url().host(), address.protocols());
             }
             ······
         }
       
       
         @Override public void configureTlsExtensions(
             SSLSocket sslSocket, String hostname, List<Protocol> protocols) {
           // Enable SNI and session tickets.
           if (hostname != null) {
             setUseSessionTickets.invokeOptionalWithoutCheckedException(sslSocket, true);
             setHostname.invokeOptionalWithoutCheckedException(sslSocket, hostname);
           }
       
           // Enable ALPN.
           if (setAlpnProtocols != null && setAlpnProtocols.isSupported(sslSocket)) {
             Object[] parameters = {concatLengthPrefixed(protocols)};
             setAlpnProtocols.invokeWithoutCheckedException(sslSocket, parameters);
           }
         }
       ```
  
       
  
     优化前
  
     ![1548665022002](/HTTPS握手过程详解与优化方法/1548665022002.png)
  
     优化后，减少一次证书交换、秘钥协商的请求
  
     ![1548665070317](/HTTPS握手过程详解与优化方法/1548665070317.png)
  
     
  
  2. False Start
  
     False Start是指客户端在发送Change CipherSpec Finished 同时发送应用数据(如HTTP请求)，服务端在TLS握手完成时直接返回应用数据。这样，应用数据的发送时机上未等到握手全部完成。开启FS之后，只需要一次RTT就可以开始传输数据，目前大部分浏览器默认都会支持启用，但也有一些前提条件：
  
     - 服务端和客户端支持NPN或者ALPN
     - 服务器配置支持前向安全性(ForwardSecrecy)的加密算法
  
     配置方法为服务器配置Nginx或者Apache开启。
  
     
  
  3. Certificate
  
     证书实在握手期间发送的，由于TCP初始拥塞窗口的存在，如果证书太长可能会产生额外的往返开销。如果证书没包含中间证书，大部分浏览器可以正常工作，但会暂停验证并根据子证书指定的父证书URL自己获取中间证书。这个过程会产生额外的DNS解析、建立TCP连接等开销，非常影响性能。
  
     配置证书的最佳实践：
  
     - 证书链是只包含站点证书和中间证书，不要包含根证书，也不要漏掉中间证书。
     - 减少证书大小，使用ECC（Elliptic Curve Cryptography，椭圆曲线密码学）证书。256位的ECC Key等同与3072位的RSA Key，在确保安全性的同事，体积大幅减少。
  
     
  
  4. 开启OCSP Stapling
  
     OCSP（Online Certificate Status Protocol，在线证书状态协议）是用来检验证书合法性的在线查询服务，一般由证书所属CA提供。某些客户端会在TLS握手阶段进一步协商时，实时查询OCSP接口，并在获得结果前阻塞后续流程。OCSP查询本质是一次完整的HTTP请求-响应，这中间的DNS查询、建立TCP、服务端处理等环节都可能耗费很长时间，导致最终建立TLS连接时间变得更长。
  
     而OCSP Stapling，是指服务端主动获取OCSP查询结果并随证书一起发送给客户单，从而让客户端跳过自己验证的过程，提高TLS握手效率。
  
     此过程由服务器Nginx配置开启。
  
     

#### WireShark配合tcpdump抓手机包

1. 下载tcpdump，adb push ~/Downloads/tcpdump  /data/local/tcpdump
2. chmod 777 /data/local/tcpdump
3. 进入root权限，su
4. tcpdump -p -vv -s 0 -w /sdcard/capture.pcap
5. adb pull /sdcard/capture.pcap ~/Downloads
6. Wireshark打开pcap文件



#### 名词解释

- RTT(round-trip time)
- 对称算法 DES AES
- 非对称算法RSA DSA



#### 参考链接

- [TLS 握手优化详解](https://imququ.com/post/optimize-tls-handshake.html)
- [HTTPS 原理以及优化实践](https://comsince.github.io/2017/06/14/http-over-ssl/)
- [有货APP HTTPS优化探索和实践](https://mp.weixin.qq.com/s/1YRhXOs61u8_fRU1wMaFyA?utm_source=androidweekly.cn&utm_medium=website)

