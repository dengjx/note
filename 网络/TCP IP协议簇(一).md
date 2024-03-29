# TCP/IP协议簇(一) HTTP简介、请求方法与响应状态码
## 一、TCP/IP协议簇简述
在总结HTTP和HTTPS之前，先简单的回顾一下TCP/IP协议簇。TCP/IP不单单指的就是TCP和IP这两个协议，而是指的与其相关的各种协议。比如HTTP, FTP, DNS, TCP, UDP, IP, SNMP等等都属于TCP/IP协议簇的范畴。

### 1.1 TCP/IP协议的分层
TCP/IP协议簇是分层管理的，在OSI标准中可以分为7层（应用层、表示层、会话层、传输层、网络层、数据链路层、物理层，可记为：应表会传网数物），因为OSI标准较为复杂，因此这里我们使用TCP/IP协议簇的四层体系结构（应用层、传输层、网络层、网络接口层/链路层）。我们在下面简单介绍一下这四层：

+ **应用层**：该层是面向用户的一层，也就是说用户可以直接操作该层，该层决定了向用户提供应用服务时的通信活动。本文要总结的HTTP（HyperText Transfer Protocol：超文本传输协议）就位于该层。我们经常使用的FTP(File Transfer Protocol: 文件传输协议)和DNS (Domain Name System: 域名系统)都位于该层。FTP简单的说就是用来文件传输的。而DNS则负责域名解析的，通过DNS可以将域名（比如：www.cnblogs.com）与IP地址（201.33.xx.09）进行相互的转换。在7层中，又将该层分为：应用层、表示层和会话层。
+ **传输层**：应用层的下方是传输层，应用层会将数据交付给传输层进行传输。TCP(Transmission Control Prococol:传输控制协议)和UDP(User Data Protocol: 用户数据协议)位于该层，当然见名知意，该层是用来提供处于网络连接中的两台计算机直接的数据传输的。TCP建立连接是需要三次握手来确认连接情况，而UDP则没有三次握手的过程。稍后会介绍。
+ **网络层**：传输层的下方是网络层，网络层用来处理在网络上流动的数据包，IP(Internet Protocol: 网际协议)就位于这层。该层负责在众多网络线路中选择一条传输线路。当然这个选择传输线路的过程需要IP地址和MAC地址的支持。
+ **网络接口层(链路层)**：在7层协议中，将网络接口层分为数据链路层和物理层。该部分主要是用来处理网络的硬件部分，我们常说的NIC（Net Work Card），也就是网卡就位于这一部分，当然光纤也是网络接口层的一部分。

![img-9](https://note.youdao.com/yws/api/personal/file/8A1387EBF9A34D8AAC02B9067856D6CA?method=download&shareKey=f81904d11b076906a18459fdf8043b0d)

下图就是这四层协议在数据传输过程中的工作方式。下面这张图还是相当直观的。在发送端是**应用层-->链路层**这个方向的封包过程，每经过一层都会增加该层的头部。而接收端则是从**链路层-->应用层**解包的过程，每经过一层则会去掉相应的首部。

![img-10](https://note.youdao.com/yws/api/personal/file/A764D964C49D4F2E8F3D80243562F5D5?method=download&shareKey=b8bf730e34a20ce5f51ff7cff9cb743e)

### 1.2 TCP协议的三次握手
TCP协议位于传输层，为了确保传输的可靠性，TCP协议在建立连接时需要三次握手（Three-way handshaking）。下方这个简图就是TCP协议建立连接时三次握手的过程。

+ 第一次握手：发送端发送一个带SYN(Synchronize)标志的数据包给接收端，用于询问接收端是否可以接收。如果可以，就进行第二次握手。
+ 第二次握手：接收端回传给发送端一个带有SYN/ACK(Acknowledgement)的数据包，给发送端说，我收到你给我发送的SYN标志了，我再给你传一个ACK标志，你能收到吗？如果发送端收到了SYN/ACK这个数据包，就可以确认接收端收到了之前发送的SYN, 然后进行第三次握手。
+ 第三次握手：发送端会给接收端发送一个带有ACK标志的数据包，告诉接收端我可以收到你给我发送的SYN/ACK标志。接收端如果收到了这个来自客户端的ACK标志，就意味着三次握手完成，连接建立，就可以开始传输数据了。

![三次握手](https://note.youdao.com/yws/api/personal/file/2CCD1ECDCBFF4F5FAE3A46685D7E87AB?method=download&shareKey=57f408efb447f49bb05e9d0eb893a06a)

## 二、HTTP报文结构
HTTP协议全称是HyperText Transfer Protocol，即超文本传输协议，用户客户端和服务器之前的通信，目前普遍使用版本为HTTP/1.1。协议本质上就是规范，我们之前提到过的“面向接口”编程，其实就是“面向协议”编程。先定义好类的协议，也就是接口，相关类都遵循该协议，这样一来我们就规范了这些类的调用方式。而HTTP协议是规范客户端和服务器之间通信的协议。也就是说所有的客户端或者服务器都遵循了HTTP这个通信协议，那么也就是意味着他们对外传输数据的接口是一直的，就可以在其中间连接上管道，这样一来就可以进行传输了。

这些协议就是接口，有着共同的通信协议，多个端就可以相互通信。采用相同的协议，就是便于个个设备之间进行沟通交流。HTTP协议的作用如下所示。

![img-11](https://note.youdao.com/yws/api/personal/file/4B98CA4DF91646E888A512D91094E65F?method=download&shareKey=ce184f7baee0d0d921f69ce872457ca9)

HTTP协议的作用是用来规范通信内容的，在HTTP协议中可以分为请求报文和响应报文。顾名思义，请求报文是请求方发出的信息，而响应报文是响应端收到请求后响应的内容。接下来我们就来看看请求报文和响应报文的整体结构。

### 2.1 请求报文（Request Message）结构
下方是请求报文的整体结构。请求报文主要分为两大部分，一个是请求头（Request Headers）另一个是请求体（Request Body）。这两者之间由空行分割。在请求头中又分为请求行（Request Line），请求头部字段，通用头部字段和实体头部字段等，这个稍后会详细介绍。下方就是请求报文的结构。

![img-12](https://note.youdao.com/yws/api/personal/file/6B41554E4026469085127DF105FF8964?method=download&shareKey=70cb2a5e9b466ce03c7b6853a312a3bc)

下方这个截图就是请求博客园某个页面时的Request Headers。在请求行中的第一个“GET”是当前请求的方法，稍后会做介绍。中间的就是请求资源的路径，最后一个HTTP/1.1就是当前使用请求协议及其版本。下方这些就是请求头了，稍后会对常用的请求头进行解说。而请求体就是你往服务端传输的数据，比如json等。

![](https://images2015.cnblogs.com/blog/545446/201612/545446-20161229150242351-1635943589.png)

### 2.2 响应报文（Response Message）结构
聊完请求报文，接下来我们来聊聊响应报文，响应报文的结构与请求报文的结构类似，也分为报文头和报文体。下方就是响应报文的结构图。响应头（Response Headers）分为状态行（State Line），响应头部字段，通用头部字段、实体头部字段等。响应头与响应体中间也是有空行进行分割的。

![](https://images2015.cnblogs.com/blog/545446/201612/545446-20161229151043757-620968303.png)

下方截图就是上述请求报文发出后的响应头，响应体就是对于的HTML等前端资源了。在响应头中，第一行就是状态行，“HTTP/1.1”表示使用的HTTP协议的1.1版本，状态200表示响应成功，"OK"则是状态原因短语。

![](https://images2015.cnblogs.com/blog/545446/201612/545446-20161229151433679-2036356360.png)

## 三、HTTP的请求方法以及响应状态码
上面在介绍请求报文中提到的“GET”就是请求请求方法，而在响应报文中提到的“200”状态码，就是稍后要聊的响应状态码。请求方法和响应状态码在HTTP协议中算是比较重要的内容了。

### 3.1 请求方法
接下来我们要聊的请求方法有GET、POST、PUT、HEAD、DELETE、OPTIONS、TRACE、CONNECT。当然上述方法是基于HTTP/1.1的，HTTP/1.0中独有的方法就不说了。

+ GET----获取资源
    + GET方法一般用来从服务器上获取资源的方法。服务器端接到GET请求后，就会明白客户端是要从服务器端获取相应的资源，然后就会根据请求报文中相应的参数，将需要的资源返回给客户端。使用GET方式的请求，传输的参数是拼接在URI上的。
+ POST----数据提交
    + POST方法一般用于表单提交，将客户端的数据塞到请求体中发送给服务器端。
+ PUT----上传文件
    + PUT方法主要用来上传文件，将文件内容塞到请求报文体中，传输给服务器。因为HTTP/1.1的PUT方法自身不带验证机制，所以任何人都可以上传文件，存在安全性，所以上传文件时不推荐使用。但是之前我们在设计接口使用REST标准时，可以使用PUT来做相应内容的更新。
+ HEAD----获取响应报文头
    + 响应端收到HEAD请求后，只会返回相应的响应头，不会返回响应体。
+ DELETE----删除文件
    + DELETE用于删除URI指定的资源，与PUT一样，自身也是不带验证机制的，不过在REST标准中可以用来做相应API的删除功能。
+ OPTIONS----查询支持的方法
    + OPTIONS方法是用来查询服务器可对那些请求方法做出相应，返回内容就是响应端所支持的方法。
+ TRACE----追踪路径
    + TRACE方法可追踪请求经过的代理路径，在发送请求时会为Max-Forwards头部字段填入数字，每经过一个代理中转Max-Forwards的值就会减一，直至Max-Forwards为零后，才会返回200。因为该方法易引起XST(Cross-Site Tracing，跨站追踪)攻击，所以不常用呢。
+ CONNECT----要求用隧道协议连接代理
    + CONNECT方法要求在与代理服务器通信时建立隧道，实现用隧道协议进行TCP通信。主要使用SSL(Secure Sockets Layer, 安全套接层)和TLS(Transport Layer Security, 传输安全层)协议将通信内容进行加密后经网络隧道传输。

### 3.2 响应状态码
聊完请求方法后，接下来我们来聊聊HTTP协议的响应状态码。顾名思义，响应状态码是用来标志HTTP响应状态的，响应状态由响应状态码和响应原因短语构成，当然状态码有很多中，本部分就挑出来常用的状态码进行讨论。下方是响应状态码可以分为的几大类：

+ 1xx ---- Informational（信息性状态码），表示接受的请求正在处理。
+ 2xx ---- Success (成功)，表示请求正常处理完毕。
+ 3xx ---- Redirection (重定向)，表示要对请求进行重定向操作，当然其中的304除外。
+ 4xx ---- Client Error (客户端错误)，服务器无法处理请求。
+ 5xx ---- Server Error (服务器错误)，服务器处理请求时出错。

上面是响应状态码的整体分类，接下来介绍一些常用的响应状态码。

+ 200 OK : 表示服务端正确处理了客户端发送过来的请求。
+ 204 No Content : 表示服务端正确处理请求，但没有报文实体要返回。
+ 206 Partial Content ：表示服务端正确处理了客户端的范围请求，并按照请求范围返回该指定范围内的实体内容。
+ 301 Moved Permanently：永久性重定向，若之前的URI保存到了书签，则更新书签中的URI。
+ 302 Found：临时重定向，该重定向不会变更书签中的内容。
+ 303 See Other：临时重定向，与302功能相同，但是303状态吗明确表示客户端应当采用GET方法获取资源。
+ 304 Not Modified: 资源未变更，该状态码与重定向并没有什么关系，当返回该状态码时，告诉客户端请求的资源并没有更新，响应报文体中并不会返回所请求的内容。
+ 400 Bad Request： 错误请求，表示请求报文中包含语法错误。
+ 401 Unauthorized：请求未认证，表示此发送的请求需要客户端进行HTTP认证（稍后会提到）。
+ 404 Not Found：找不到相应的资源，表示服务器找不到客户端请求的资源。
+ 500 Internal Server Error：服务器内部错误，表示服务器在处理请求时出现了错误，发生了异常。
+ 503 Service Unavailable：服务不可用，表示服务器处于停机状态，无法处理客户端发来的请求。






