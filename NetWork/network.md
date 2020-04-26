## 网络编程

### 目录

* [1. 网络分层](https://github.com/NieJianJian/AndroidNotes/blob/master/NetWork/network.md#1-网络分层)
* [2. TCP和UDP](https://github.com/NieJianJian/AndroidNotes/blob/master/NetWork/network.md#2-tcp和udp)
  * [2.1 TCP](https://github.com/NieJianJian/AndroidNotes/blob/master/NetWork/network.md#21-tcp)
  * [2.2 UDP](https://github.com/NieJianJian/AndroidNotes/blob/master/NetWork/network.md#22-udp)
* [3. HTTP](https://github.com/NieJianJian/AndroidNotes/blob/master/NetWork/network.md#3-http)
  * [3.1 HTTP简介](https://github.com/NieJianJian/AndroidNotes/blob/master/NetWork/network.md#31-http简介)
  * [3.2 HTTP请求报文](https://github.com/NieJianJian/AndroidNotes/blob/master/NetWork/network.md#32-http请求报文)
  * [3.3 HTTP响应报文](https://github.com/NieJianJian/AndroidNotes/blob/master/NetWork/network.md#33-http响应报文)
  * [3.4 HTTP消息报头](https://github.com/NieJianJian/AndroidNotes/blob/master/NetWork/network.md#34-http消息报头)
  * [3.5 HTTP请求方式](https://github.com/NieJianJian/AndroidNotes/blob/master/NetWork/network.md#35-http请求方式)
  * [3.6 HTTP拓展知识](https://github.com/NieJianJian/AndroidNotes/blob/master/NetWork/network.md#36-http拓展知识)
* [4. HttpClient和HttpURLConnection](https://github.com/NieJianJian/AndroidNotes/blob/master/NetWork/network.md#4-httpclient和httpurlconnection)
  * [HttpClient](https://github.com/NieJianJian/AndroidNotes/blob/master/NetWork/network.md#41-httpclient)
  * [HttpURLConnection](https://github.com/NieJianJian/AndroidNotes/blob/master/NetWork/network.md#42-httpurlconnection)

***

***

***

### 1. 网络分层

业内普遍的分层方式有两种，OSI七层模型和TCP/IP四层模型

<table>
  	<tr align="center">
      	<th>OSI(理论上的标准)</th>
      	<th>TCP/IP(事实上的标准)</th>
      	<th>对应网络协议</th>
    </tr>
  	<tr align="center">
      	<td>应用层</td>
      	<td rowspan="3">应用层</td>
      	<td>HTTP、Telnet、FTP、NFS、SMTP</td>
    </tr>
  	<tr align="center">
      	<td>表示层</td>
      	<td>加密、ASCII</td>
    </tr>
  	<tr align="center">
      	<td>会话层</td>
      	<td>RPC、SQL</td>
    </tr>
  	<tr align="center">
      	<td>传输层</td>
      	<td>传输层</td>
      	<td>TCP、UDP</td>
    </tr>
  	<tr align="center">
      	<td>网络层</td>
      	<td>网络层</td>
      	<td>IP、IPX</td>
    </tr>
  	<tr align="center">
      	<td>数据链路层</td>
      	<td rowspan="2">数据链路层</td>
      	<td>HDLC、FDDI</td>
    </tr>
  	<tr align="center">
      	<td>物理层</td>
      	<td>Rj45、802.3</td>
    </tr>
</table>

* **物理层**

  该层负责比特流节点间的传输，即负责物理传输。通俗来讲就是把计算机连接起来的物理手段。

* **数据链路层**

  该层控制网络层和物理层之间的通信，主要功能是如何在不可靠的物理线路上进行数据的可靠传递。为了保证传输，从网络层接收到的数据被分割成特定的可被物理层传输的帧。

* **网络层**

  该层决定如何将数据从发送方路由到接收方。网络层综合考虑发送优先权、网络拥塞程度、服务质量以及可选路由的花费来决定一个网络中的节点A到另一个网络的节点B的最佳路径。

* **传输层**

  该层为两台主机上的应用程序提供端到端的通信。相比之下，网络层的功能是建立主机到主机的通信。

* **应用层**

  应用程序收到传输层的数据后，进行解读。解读必须事先定好格式，而应用层就是规定应用程序的数据格式。

***

***

***

### 2. TCP和UDP

TCP和UDP的比较

|          | UDP                                                          | TCP                                    |
| :------- | :----------------------------------------------------------- | :------------------------------------- |
| 是否连接 | 无连接                                                       | 面向连接                               |
| 是否可靠 | 不可靠传输，不使用流量控制和拥塞控制<br />无法保证数据正确传输 | 可靠传输，使用流量控制和拥塞控制       |
| 连接个数 | 支持一对一，一对多，多对一和多对多交互通信                   | 只能是一对一通信                       |
| 传输方式 | 面向报文                                                     | 面向字节流                             |
| 首部开销 | 首部开销小，仅8字节                                          | 首部最小20字节，最大60字节             |
| 重传机制 | 没有                                                         | 超时重传                               |
| 其他     | 稳定性差，效率高                                             | 稳定性高，效率差                       |
| 适用场景 | 适用于实时应用（IP电话、视频会议、直播等）                   | 适用于要求可靠传输的应用，例如文件传输 |

#### 2.1 TCP

* **TCP报文**

  ![TCP报文段的首部格式](https://raw.githubusercontent.com/NieJianJian/AndroidNotes/master/Picture/tcpsegementheaderformat.jpeg)

  * **源端口**：标识报文的返回地址。

  * **目的端口**：指明接收方计算机上的应用地址。

  * **TCP报头中的源端口号和目的端口号同IP数据报中的源IP与目的IP唯一确定一条TCP连接**。

  * 序号和确认号：TCP可靠传输的关键部分。

    * **序号**：是本报文段发送的数据组的第一个字节的序号。在TCP传送的流中，每一个字节一个序号。比如一个报文段的序号为300，此报文段数据部分共有100字节，则下一个报文段的序号为400。所以**序号确保了TCP传输的有序性**。
    * **确认号**：即ACK，指明下一个期待收到的字节序号，表明该序号之前的所有数据已经正确无误的收到。确认号只有当ACK标志为1时才有效。比如建立连接时，SYN报文的ACK标志位为0。

  * **数据偏移／首部长度**：4bits。由于首部可能含有可选项内容，因此TCP报头的长度是不确定的，报头不包含任何任选字段则长度为20字节，4位首部长度字段所能表示的最大值为1111，转化为10进制为15，15*32/8 = 60，故报头最大长度为60字节。首部长度也叫数据偏移，是因为首部长度实际上指示了数据区在报文段中的起始偏移值。

  * **保留**：为将来定义新的用途保留，现在一般置0。

  * **控制位**：URG  ACK  PSH  RST  SYN  FIN，共6个，每一个标志位表示一个控制功能。

    * **URG**：紧急指针标志，为1时表示紧急指针有效，为0则忽略紧急指针。

    * **ACK**：确认序号标志，为1时表示确认号有效，为0表示报文中不含确认信息，忽略确认号字段。（建立连接后所有发送的报文ACK必须为1）

    * **PSH**：push标志，为1表示是带有push标志的数据，指示接收方在接收到该报文段以后，应尽快将这个报文段交给应用程序，而不是在缓冲区排队。

    * **RST**：重置连接标志，用于重置由于主机崩溃或其他原因而出现错误的连接。或者用于拒绝非法的报文段和拒绝连接请求。

    * **SYN**：同步序号，用于建立连接过程，在连接请求中，SYN=1和ACK=0表示该数据段没有使用捎带的确认域，而连接应答捎带一个确认，即SYN=1和ACK=1。

    * **FIN**：finish标志，用于释放连接，为1时表示发送方已经没有数据发送了，即关闭本方数据流。

  * **窗口**：滑动窗口大小，用来告知发送端接受端的缓存大小，以此控制发送端发送数据的速率，从而达到**流量控制**。窗口大小是一个16bit字段，因而窗口大小最大为**65535**。

  * **校验和**：奇偶校验，此校验和是对整个的 TCP 报文段，包括 TCP 头部和 TCP 数据，以 16 位字进行计算所得。由发送端计算和存储，并由接收端进行验证。

  * **紧急指针**：只有当 URG标志置 1 时紧急指针才有效。紧急指针是一个正的偏移量，和顺序号字段中的值相加表示紧急数据最后一个字节的序号。 TCP 的紧急方式是发送端向另一端发送紧急数据的一种方式。

  * **选项和填充**：最常见的可选字段是最长报文大小，又称为MSS（Maximum Segment Size），每个连接方通常都在通信的第一个报文段（为建立连接而设置SYN标志为1的那个段）中指明这个选项，它表示本端所能接受的最大报文段的长度。选项长度不一定是32位的整数倍，所以要加填充位，即在这个字段中加入额外的零，以保证TCP头是32的整数倍。

  * **数据部分**： TCP报文段中的数据部分是可选的。在一个连接建立和一个连接终止时，双方交换的报文段仅有 TCP 首部。如果一方没有数据要发送，也使用没有任何数据的首部来确认收到的数据。在处理超时的许多情况中，也会发送不带任何数据的报文段。

* **TCP连接过程（三次握手**）

  > seq：Sequence Number，序列号，是数据包本身的序列号
  >
  > ACK：Acknowledgment，确认值，1表示连接。
  >
  > ack：Acknowledgment Number，确认编号，是期望对方继续发送的那个数据包的序列号，一般为上一次远端主机发来的seq+1。

  * 第一次握手：客户端发送连接请求报文，设置SYN=1、ACK=0、seq=x，接下来客户端进入SYN_SENT状态，等待服务端确认。
  * 第二次握手：服务端收到连接请求，对SYN报文段进行确认，设置SYN=1、ACK=1、ack=x+1、seq=y，服务端将上述信息放入SYN+ACK报文段，一并发送给客户端，服务端进入SYN_RCVD状态。
  * 第三次握手：客户端收到报文段，设置ACK=1、seq=x+1、ack=y+1，向服务端发送ACK报文段，发送完毕后，客户端和服务端都进入ESTABLISHED（TCP连接成功）状态，完成三次握手。

* **TCP断开连接（四次挥手**）

  * 第一次挥手：设置FIN=1、seq=u，发送FIN报文段，客户端进入FIN_WAIT_1状态，表示客户端没有数据要发送给服务端了。
  * 第二次挥手：服务端收到客户端的FIN报文段，设置ACK=1、ack=u+1、seq=v，并发送给客户端。
  * 第三次挥手：服务端向客户端发送FIN报文段，请求关闭连接，同时服务端进入LAST_ACK状态。
  * 第四次挥手：客户端收到服务端FIN报文段，向服务端发送ACK报文段，然后客户端进入TIME_WAIT状态。服务端接收到客户端的ACK报文段，就关闭连接。此时，客户端等待2MSL（最大报文生存时间）后依然没有收到回复，说明服务端已经正常关闭，客户端也关闭连接。

* **TCP主要特点**

  * 面向连接，是指发送数据前建立连接，保证数据传输的可靠性。
  * 单播传输，仅支持点对点的数据传输。
  * 可靠传输，每次TCP传输都有序号和确认号，保证了数据的有序传输，如果延时没收到，数据将被重传。
  * 拥塞控制，当网络出现拥塞时，TCP能够减少向网络注入数据的速率和数量。
  * 全双工通信，允许通信双方任何时候都能发送数据，也允许双方独立关闭自己的通信，也就是半关闭。

* **【问题1】：如果大量连接，每次都连接、关闭会造成性能低下怎么办**？

  HTTP有一种keepalive connections的机制，可以在传输数据后仍然保持连接，当客户端需要再次获取数据时，直接使用刚刚空闲下来的连接无需再次握手。

* **【问题2】：建立连接为什么要三次握手（两次确认）**？

  个人理解：双方都需要确认自己发送的序列号有没有被对方收到，所以三次正好。

  TCP的三次握手也主要是防止收到已过期的连接而再次传到被连接的主机。TCP具有超时重传机制，如果发送的连接请求由于某种原因没有及时到底，导致超时，客户端会重新发送一个连接请求，之后突然之前没有到底的连接请求又传到了服务端，如果是两次握手的话，服务端只要收到客户端的连接请求，就建立连接，这是不合理的。

* **【问题3】：为什么连接三次，断开要四次**？

  由于TCP是双工的通信机制，通信双方都可以独立关闭自己的通道，也就是半关闭。每次一个方向上的关闭都需要FIN和ACK两次握手，由于TCP对网络报文是迅速响应的，当服务端收到客户端FIN请求时，服务端会理解回复客户端一个ACK报文段，表示"表示我收到你的断开请求了，但是我还没发送完，你得再等等"，当服务端发送数据完成了，就会发送一个FIN报文段到客户端，表示我发完了，可以关闭了。

* **【问题4】：为什么客户端要进入TIME_WAIT，并等待2MSL的时间**？

  持续2MSL，一个数据包在网络中的最长生存时间。是为了防止客户端最后一次回复的ACK丢失，服务端会在超时时间到来时，重传一个FIN包。

***

#### 2.2 UDP

* **UDP报文**

  ![UDP报文格式](https://raw.githubusercontent.com/NieJianJian/AndroidNotes/master/Picture/udpsegementheaderformat.png)

  

* **UDP主要特点**

  * 面向无连接，不需要三次握手建立连接，想发就发，只是数据报文的搬运工，不会对报文进行拆分和拼接操作。发送数据时，应用层将数据传递到传输层的UDP协议，UDP只对数据增加一个UDP标识而已。
  * 单播、多播，可以一对一、一对多、多对一的方式。
  * 不可靠性，无连接，想发就发，不会备份数据，不关心接收方是否正确接收数据，没有拥塞控制，容易丢包。

***

***

***

### 3. HTTP

#### 3.1 HTTP简介

HTTP是一个应用层协议，它通过TCP实现了可靠的数据传输。

* HTTP协议的主要特点

  * 支持C/S（客户端/服务器）模式
  * 简单快速：客户向服务器请求服务时，只需要传送请求方法和路径。
  * 灵活：HTTP允许传输任意类型的数据对象。正在传输的类型由Content-Type加以标记。
  * 无连接：无连接的含义是每次连接只处理一个请求。
  * 无状态：HTTP协议是无状态协议，无状态是指协议对于事务处理没有记忆的能力。缺少状态意味着如果后续处理需要前面的信息，则它必须重传，这样可能导致每次连接传送的数据量增大；而另一方面，在服务器不需要先前信息时它的应答速度就比较快。

* HTTP URL的格式如下：

  ```
  http://host[":port"][abs_path]
  ```

  * `http`表示通过HTTP协议来定位网络资源；
  * `host`表示合法的Internet主机域名或者IP地址；
  * `port`指定一个端口号，为空则使用默认端口80；
  * `abs_path`指定请求资源的URI（Web上任意的可用资源）。

* HTTP有两种报文，分别是请求报文和相应报文。

***

#### 3.2 HTTP请求报文

HTTP请求报文由请求行、请求报头、空行和请求数据4个部分组成。

![图：HTTP请求报文的一般格式](https://raw.githubusercontent.com/NieJianJian/AndroidNotes/master/Picture/httprequestmessageformat.jpg)

* **请求行**

  请求行由请求方法、URL字段和HTTP协议的额版本组成，格式如下：

  ```
  Method Request-URI HTTP-Version CRLF
  ```

  * `Method`表示请求方法，如GET、PSOT、PUT等；
  * `Request-URI`是一个统一资源标识符；
  * `HTTP-Version`表示请求的HTTP协议版本；
  * `CRLF`表示回车符和换行（CR和LF不允许单独出现）。

  访问百度的请求行如下：

  ```
  GET http://www.baidu.com HTTP/1.1
  ```

* **请求报头**

  在请求行只会有0个或者多个请求报头，每个请求报头都包含一个名字和一个值，它们之间用英文冒号":"分隔。如`Connection:Keep-Alive`。

* **请求数据**

  请求数据中包含了要发送给Web服务器的数据。POST、PUT该部分为可选部分，GET和DELETE无该部分。请求行和请求报头都是结构化文本形式，而数据主体可以包含任意的二进制数据（如图片、视频、音频、软件程序）。当然，主体也可以包含文本。

***

#### 3.3 HTTP响应报文

HTTP响应报文由状态行、响应包头、空行、响应正文4个部分组成。

![图：HTTP响应报文的一般格式](https://raw.githubusercontent.com/NieJianJian/AndroidNotes/master/Picture/httpresponsemessageformat.png)

* **状态行**

  状态行格式如下：

  ```
  HTTP-Version Status-Code Reason-Phrase CRLF
  ```

  * `HTTP-Version`表示服务器的HTTP协议的版本；
  * `Status-Code`表示服务器发回的响应状态码；
  * `Reason-Phrase`表示状态码的文件描述。

  状态行例子如下：

  ```
  HTTP/1.1 200 OK
  ```

* **状态码**

  状态码由3个数字组成，第一个数字定义了一个响应的类别，且有以下5种可能取值：

  * 100~199：指示信息，表示请求已经收到，继续处理。
  * 200~299：请求成功，请求已经被成功接受并处理。
  * 300~399：重定向，要完成请求必须进行更进一步操作。
  * 400~499：客户端错误，请求有语法错误或请求无法实现。
  * 500~599：服务器错误，服务器不能实现合法的请求。

  常见状态码、状态描述如下：

  * 200 OK：客户端请求成功。
  * 400 Bad Request：客户端请求有语法错误，服务器无法理解。
  * 401 Unauthorized：请求未经授权，这个状态码必须和WWW-Authenticate报头域一起使用。
  * 403 Forbidden：服务器收到请求，但是拒绝提供服务。
  * 404 Not Found：请求资源不存在。比如输入错误的URL。
  * 500 Internal Server Error：服务器内部错误，无法完成请求。
  * 503 Server Unavaliable：服务器当前不能处理客户端的请求，一段时间后可能回复正常。

****

#### 3.4 HTTP消息报头

消息报头分为通用报头、请求报头、响应报头、实体报头等。由键值对组成，用冒号":"分隔。

| 报头类型 | 作用                                                         |
| -------- | ------------------------------------------------------------ |
| 通用报头 | 既可以出现在请求报头、也可以出现在响应报头                   |
| 请求报头 | 通知服务器关于客户端的请求信息                               |
| 响应报头 | 用于服务器传递自身信息的报头                                 |
| 实体报头 | 定义被传送资源的信息，描述主体的长度和内容，或者资源本身，可用于请求和响应 |

* 通用报头
  * Date：表示消息产生的日期和时间。
  * Connection：允许发送指定连接的选项。
  * Cache-Control：用于指定缓存指令，缓存指令时单向的（响应中出现的缓存指令在请求中未必出现），且独立的（一个消息的缓存指令不会影响到下一个消息处理的缓存机制）。
* 请求报头
  * Host：请求的主机名，允许多个域名同处于一个IP地址，即虚拟主机。
  * User-Agent：发送请求的浏览器类型、操作系统等信息。
  * Accept：客户端可以识别的内容类型列表，用于指定客户端接收哪些类型的信息。
  * Accept-Encoding：客户端可识别的数据编码。
  * Accept-Language：表示客户端所支持的语言类型。
  * Connection：允许客户端和服务器指定与请求/响应连接有关的选项。如Keep-Alive则表示保持连接。
  * Transfer-Encoding：告知接收端为了保证报文的可靠传输，对报文采取了什么编码方法。
* 响应报头
  * Location：用于重定向接收者到一个新的为止，常用于更换域名的时候。
  * Server：包含服务器用来处理请求的系统消息，与User-Agent请求报头是相对应的。
* 实体报头
  * Content-Type：请求数据的格式（媒体类型）。
  * Content-Length：实体正文的消息长度。
  * Content-Language：描述资源所用的自然语言。
  * Content-Encoding：用作媒体类型的修饰符。它的值指示正文附加内容的编码，因而要获得Content-Type报头域种所引用的媒体类型，必须采用响应的编码机制。
  * Last-Modified：用于指示资源的最后修改日期和时间。
  * Expires：给出响应过期的日期和时间。

***

#### 3.5 HTTP请求方式

* GET

  作用是获取服务器中的某个资源。请求的参数需要放到请求的URL中。

* POST

  起初是用来向服务器传输数据的，实际上POST请求通常会用来提交HTML表单。

* PUT

  与GET获取服务器读取资源相反，PUT方法会向服务器写入资源。

* DELETE

  请求服务器删除请求URL所指定的资源。客户端无法保证该操作一定会被执行。因为HTTP允许服务器在不通知客户端的情况下撤销请求。与GET一样，参数放在请求的URL中。

* HEAD

  与GET方法类似，但是服务器在响应中只返回首部，不会返回实体的主体部分。可以在不获取资源的情况下，对资源的首部进行检查

* TRACE

  客户端发起一个请求，这个请求可能要穿过防火墙、代理、网关或者其他一些应用程序。每个中间节点都可能修改原始的HTTP请求。TRACE方法允许客户端在最终将请求发送给服务器时，看看它变成了什么样子。

  * 可以用于诊断，用于验证请求是否如愿穿过了请求/响应链
  * 可以用于查看代理和其他应用程序对用户请求所产生的效果。

* OPTIONS

  OPTIONS方法请求Web服务器告知其所支持的各种功能。

* CONNECT

  要求与代理服务器通信时建立隧道，实现用隧道协议进行TCP通信。

* GET和DELETE的URL最大长度为1024字节，也就是1KB。

* PUT和POST它们的报文格式一般是表单形式，也就是说这两个请求方式的参数存储在报文的请求数据上。

***

#### 3.6 HTTP拓展知识

* **SSL**

  Secure Sockets Layer ，是安全套接字协议。为了解决HTTP明文传输的不安全性（虽然POST提交的数据在报体中看不到，但是可以通过抓包工具窃取到）。SSL是基于HTTP之下TCP之上的一个协议层（也就是传输层和应用层之间），是基于HTTP标准并对TCP传输数据时进行加密。

* **HTTPS**

  Hyper Text Transfer Protocol over SecureSocket Layer，是安全层套接字层的超文本传输协议。

  HTTPS = HTTP + SSL。

* **TLS**

  Transport Layer Security，是传输层安全协议。SSL协议是TLS协议的前身。

* **SOCKS**

* **DNS**

  域名系统（Domain Name System缩写DNS，Domain Name被译为域名）是因特网的一项核心服务，它作为可以将域名和IP地址相互映射的一个分布式数据库，能够使人更方便的访问互联网，而不用去记住能够被机器直接读取的IP数串。

  **作用**：用来解析域名。互联网中其实不存在www.xxx.com这样域名方式，而是255.255.255.255这样的IP地址，访问域名www.xxx.com，DNS服务器会将域名映射到对应的IP地址上，从而完成域名解析。（域名方便记忆）

* **DNS寻址过程**

  这里用浏览器访问www.baidu.com来举例。

  * 先从浏览器缓存中查找是否有域名对应的ip地址。如果找到，域名解析结束。
  * 浏览器缓存未找到，则到操作系统中去查找，比如C盘的hosts文件。
  * 如果操作系统找不到的话，就访问本地DNS服务器（本地DNS应该是指电脑里设置的IPv4中填写的DNS地址，或者是DHCP自动分配的。连接路由器的话，就是路由器分配的DNS地址。也比如直接连接网络就是运营商的DNS），绝大多数到这一步就结束了。
  * 本地DNS找不到，就去请求根DNS服务器（域名解析起点）。
  * 根DNS服务器根据请求的域名的根域部分（比如.com），返回相应的下一级DNS服务器的地址。
  * 本地DNS访问下一级DNS服务器，这样递归的方法一级一级查找，直到找到相应的IP地址并返回。
  * 本地DNS将查询到的ip地址返回给浏览器。
  * 浏览器根据得到的ip地址去访问目标主机，完成访问。

* **代理**

  客户端和服务器的"中间人"，进行接收和转发。

* **网关**

  网关的工作机制和代理十分相似。而网关能使通信线路上的服务器提供非 HTTP 协议服务。

  假设：IP=10.1.1.2，Mask = 255.255.255.0，Gateway = 10.1.1.1

  * 自己和自己通信（10.1.1.2）：流量在主机内溜达，不会触碰网线
  * 与广播域内其他主机（10.1.1.x）通信：通过ARP广播发现其它主机的MAC，这些流量在主机间穿梭，不会到达网关。
  * 与主机外通信：访问外网会通过网关。

* **隧道**（**Tunnel**）

  在相隔甚远的客户端和服务器之间进行中转，并保持双方通信连接的应用程序。隧道本身不去解析HTTP请求，请求保持原样中转给之后的服务器。

* **HTTP和HTTPS的区别**

  * HTTP明文传输，不安全；HTTPS加密协议，是安全的。
  * HTTP不需要整数，HTTPS需要CA证书。
  * HTTP标准端口是80，HTTPS标准端口是443。
  
* **HTTPS加密**

  **HTTPS加密采用的是对称加密和非对称加密结合的方式**。

  * 加密方式：
    * 对称加密：加密解密都是同一把密钥。
    * 非对称加密：密钥成对出现，分为公钥和私钥，传输双方均有一对自己的密钥，公钥加密通过私钥进行解密。传输双方都将自己的公钥发给对方，对方用公钥进行加密，自己用私钥进行解密。
  * 加密方式区别
    * 对称加密算法简单，加密速度快，非对称加密算法复杂，加密速度慢。
    * 对称加密要将密钥暴露，和明文传输没区别。
    * 非对称加密算法复杂，耗时是对称加密的数千倍。
  * HTTPS加密过程
    * 浏览器使用HTTPS的URL访问服务器，建立SSL链接；
    * 服务器收到SSL链接，发送非对称加密的公钥A返回给浏览器；
    * 浏览器随机生成对称加密的密钥B；
    * 浏览器使用公钥A，对自己生成的密钥B进行加密，得到密钥C；
    * 浏览器将密钥C，发送给服务器；
    * 服务器用私钥D对接受的密钥C进行解密，得到对称加密钥B；
    * 浏览器和服务器之间可以用密钥B作为对称加密密钥进行通信。

***

***

***

### 4. HttpClient和HttpURLConnection

#### 4.1 HttpClient

HttpClient的一般使用步骤如下：

1. 使用DefaultHttpClient类实例化HttpClient对象；
2. 创建HttpGet或HttpPost对象，并把将要请求的URL通过构造方法注入；
3. 调用execute方法发送HTTP GET 或HTTP POST请求，并返回HttpResponse对象；
4. 通过HttpResponse接口的getEntity方法返回响应信息，并进行相应的处理。

示例代码如下：

1. 实例化HttpClient

   ```java
   private HttpClient createHttpClient() {
       HttpParams params = new BasicHttpParams();
       HttpConnectionParams.setConnectionTimeout(params, 1500);
       HttpConnectionParams.setSoTimeout(params, 1500);
       HttpConnectionParams.setTcpNoDelay(params, true);
       HttpProtocolParams.setVersion(params, HttpVersion.HTTP_1_1);
       HttpProtocolParams.setContentCharset(params, HTTP.UTF_8);
       // 持续握手
       HttpProtocolParams.setUseExpectContinue(params, true);
       HttpClient client = new DefaultHttpClient(params);
       return client;
   }
   ```

2. 创建HttpGet和HttpClient，请求网络并得到HttpResponse，并进行处理

   ```java
   private void userHttpClientGet(String url) throws IOException {
       HttpGet httpGet = new HttpGet(url);
       httpGet.addHeader("Connection", "Keep-Alive");
       HttpClient httpClient = createHttpClient();
       HttpResponse httpResponse = httpClient.execute(httpGet);
       HttpEntity httpEntity = httpResponse.getEntity();
       int code = httpResponse.getStatusLine().getStatusCode();
       if (httpEntity != null) {
           InputStream inputStream = httpEntity.getContent();
           String response = converStreamToString(inputStream);
           Log.i("niejianjian","请求结果：" + response);
           inputStream.close();
       }
   }
   ```

   上面用到的converStreamToString方法将请求结果转换为String类型：

   ```java
   private String converStreamToString(InputStream is) throws IOException {
       BufferedReader reader = new BufferedReader(new InputStreamReader(is));
       StringBuffer sb = new StringBuffer();
       String line = null;
       while ((line = reader.readLine()) != null) {
           sb.append(line + "\n");
       }
       return sb.toString();
   }
   ```

   HTTP GET缺点是把请求参数作为URL的一部分来传递，而且URL的长度应该在2048之内，HTTP 1.1之后URL的长度才没有限制。

3. 使用POST请求代码如下：

   ```java
   private void userHttpClientPost(String url) throws IOException {
       HttpPost httpPost = new HttpPost(url);
       httpPost.addHeader("Connection", "Keep-Alive");
       HttpClient httpClient = createHttpClient();
       List<NameValuePair> params = new ArrayList<>();
       params.add(new BasicNameValuePair("name","xiaoxie"));
       params.add(new BasicNameValuePair("pass","123456"));
       httpPost.setEntity(new UrlEncodedFormEntity(params));
       HttpResponse httpResponse = httpClient.execute(httpPost);
       HttpEntity httpEntity = httpResponse.getEntity();
       int code = httpResponse.getStatusLine().getStatusCode();
       if (httpEntity != null) {
           InputStream inputStream = httpEntity.getContent();
           String response = converStreamToString(inputStream);
           Log.i("niejianjian","请求结果：" + response);
           inputStream.close();
       }
   }
   ```

***

#### 4.2 HttpURLConnection

HttpURLConnection的API简单，体积较小，非常适用与Android项目。HttpURLConnection的压缩和缓存机制可以有效的减少网络访问的流量，在提升速度和省点方面也起到了较大作用。

HttpURLConnection的POST请求代码如下：

```java
private HttpURLConnection getHttpURLConnection(String url) throws IOException {
    HttpURLConnection mHttpURLConnection = null;
    URL mUrl = new URL(url);
    mHttpURLConnection = (HttpURLConnection) mUrl.openConnection();
    mHttpURLConnection.setConnectTimeout(1500);
    mHttpURLConnection.setReadTimeout(1500);
    mHttpURLConnection.setRequestMethod("POST");
    mHttpURLConnection.setRequestProperty("Connection", "Keep-Alive");
    mHttpURLConnection.setDoInput(true);
    // post传递参数时，需要开启输入功能
    mHttpURLConnection.setDoOutput(true);
    return mHttpURLConnection;
}

public static void postParams(OutputStream output, List<NameValuePair> list)
        throws IOException {
    StringBuffer mStringBuilder = new StringBuffer();
    for (NameValuePair pair : list) {
        if (!TextUtils.isEmpty(mStringBuilder)) {
            mStringBuilder.append("&");
        }
        mStringBuilder.append(URLEncoder.encode(pair.getName(), "UTF-8"));
        mStringBuilder.append("=");
        mStringBuilder.append(URLEncoder.encode(pair.getValue(), "UTF-8"));
    }
    BufferedWriter writer = new BufferedWriter(new OutputStreamWriter(output, "UTF-8"));
    writer.write(mStringBuilder.toString());
    writer.flush();
    writer.close();
}
```

**注意**：一旦向输出流中写入了参数，请求方式会变成POST，即使设置的是GET方法。

***

***

***

### 参考链接

[TCP报文格式详解](https://blog.csdn.net/paincupid/article/details/79726795)

[TCP 为什么是三次握手，而不是两次或四次](https://www.zhihu.com/question/24853633)

[一文搞懂TCP和UDP](https://www.cnblogs.com/fundebug/p/differences-of-tcp-and-udp.html)

[代理，网关，隧道，有什么区别与联系](https://www.idcbest.com/idcnews/11003815.html)

[代理，网关，隧道，有什么区别与联系？](https://www.zhihu.com/question/268204483)

[什么是HTTP隧道，怎么理解HTTP隧道呢？](https://www.zhihu.com/question/21955083)

