# 主要特点

* TCP是**面向连接的运输层协议**。使用TCP协议之前，必须先建立TCP连接。在传输数据完毕后，必须释放已经建立的TCP连接
* 每条TCP连接只能有两个**端点(endpoint)**,只能是点对点，一对一
* TCP提供**可靠交付**的服务。通过TCP连接传送的数据，无差错、不丢失、不重复，并且按序到达
* TCP提供**全双工通信**。TCP允许通信双方的应用进程在任何时候都能发送数据。两端都设有发送缓存和接收缓存，用来临时存放通信数据
* **面向字节流**。**流(stream)指的是流入到进程或从进程流出的字节序列**。面向字节流的含义是：虽然应用程序和TCP的交互是一次一个数据块(大小不等)，但TCP把应用程序交下来的数据仅仅看成是一连串的无结构的字节流

TCP的连接是一条虚连接(逻辑连接)，而不是一条真正的物理连接

TCP和UDP在发送报文时所采用的方式不同，TCP并不关心应用进程一次把多长的报文发送到TCP的缓存中，而是根据对方给出的窗口值和当前网络拥塞的程度来决定一个报文段应包含多少个字节(UDP发送的报文长度是应用进程给出的)

如果应用进程传输到TCP缓存的数据块太长，TCP就可以把它划分短一些再传送，如果应用进程一次只发来一个字节，TCP也可以等待积累有足够多的字节后再构成报文段发送出去

# TCP的连接

TCP把**连接**作为**最基本的抽象**

TCP连接的端点是什么？不是主机，不是主机的IP地址，不是应用进程，也不是运输层的协议端口。TCP连接的端点叫做**套接字(socket)**。端口拼接到IP地址即构成套接字

**每一条TCP连接唯一地被通信两端的两个端点(即两个套接字)所确定**

随着互联网的发展，名词socket可表示多种不同意思，例如:

- 允许应用程序访问连网协议的**应用编程接口API**，即运输层和应用层之间的一种接口，称为socket API，简称socket
- 在socket API中使用的一个函数叫做socket
- 调用socket函数时，函数的返回值称为socket描述符，可简称为socket
- 在操作系统内核中连网协议的Berkeley实现，称为socket实现


