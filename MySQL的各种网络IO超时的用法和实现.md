# MySQL的各种网络IO超时的用法和实现

https://zhuanlan.zhihu.com/p/50739231

## **客户端C API**

在C API中调用mysql_options()来设置mysql_init() 所创建的连接对象的属性，使用这三个选项可以设置连接超时和读写超时，单位都是秒。读写超时达到后C API的查询发送和结果获取函数会返回超时错误。

MYSQL_OPT_CONNECT_TIMEOUT

MYSQL_OPT_READ_TIMEOUT

MYSQL_OPT_WRITE_TIMEOUT

也可以使用配置文件来设置连接超时和交互超时：

connect-timeout=seconds

interactive-timeout=seconds

当客户端API在向mysql server发起连接connect-timeout秒后没有收到mysql server的相应那么认为连接失败。

interactive-timeout是用于这个客户端连接的交互超时。交互超时用于在交互界面程序（比如mysql这个连接客户端程序）中设置server的会话超时时间，来赋值给wait_timeout会话变量，它通常比较大，因为使用这个交互界面的通常是人类在手动操作，而不是程序执行和发送指令。mysql server默认使用会话变量@@wait_timeout 作为会话的超时，除非在mysql_real_connect中使用CLIENT_INTERACTIVE标志来标识这个连接是用于交互操作，此时如果客户端有interactive-timeout就使用它来作为本会话的会话超时，否则使用服务器端的interactive-timeout作为会话超时设置给wait_timeout。

## **MySQL Server内部**

从下图中可以看出，服务器端关于超时的变量有很多，这里去掉了我们的TDSQL特有的非标准的MySQL/MariaDB的几个超时值。其中的connect_timeout, net_read_timeout, net_write_timeout，slave_net_timeout， interactive_timeout和wait_timeout与网络IO有关。

![](https://pic4.zhimg.com/80/v2-9edfd3f3c3392b3d27b4f31e2a44a3c7_1440w.jpg)

其中，connect_timeout 被用于在用户登录期间，也就是建立数据库连接期间，作为mysql server端的网络读写超时。这与文档上面的说法并不相同，但是代码中确实是这样子的。

net_read_timeout 和net_write_timeout是数据库会话创建好之后mysql server端使用的读写超时。如果读取或者写入操作在等待了达到超时后服务器认为客户端连接断开，执行错误处理。

而slave_net_timeout是slave的io线程使用客户端C API连接master时候，调用mysql_options()来设置MYSQL_OPT_CONNECT_TIMEOUT，MYSQL_OPT_READ_TIMEOUT，MYSQL_OPT_WRITE_TIMEOUT这三个超时选项使用的值。IO线程连接master使用的都是标准的客户端C API代码和通信协议。

最后，wait_timeout是mysqld server的默认的会话超时，如果一个数据库连接（会话）在这么长时间之后没有任何读写动作，那么这个连接被关闭。interactive_timeout是默认的交互式连接的会话超时，会设置给wait_timeout，如果客户端有自定义的值，那么那个值会被优先使用来设置给wait_timeout。

所有这些超时可以分为连接超时，读写超时和会话超时三类，下面就讲一下这些超时机制是如何实现的。

## **超时的实现方法**

在Linux的connect(), recv, send, read(), write()等系统调用中，并不可以简单地阻塞等待一段指定时间后再返回错误，而是要么把文件句柄设置为非阻塞的（使用fcntl()和O_NONBLOCK标志，或者对于recv/send()可以每次调用使用MSG_DONTWAIT）并且立刻返回，要么一直阻塞等待。所以，要实现超时还是要一点小技巧的。另外，MySQL中网络IO的代码无论是客户端C API的网络IO功能还是服务器内部使用的网络通信功能，都是同一份代码实现，因此下文不需要区分客户端和服务器端。

## **连接超时**

相关函数：vio_socket_connect()，vio_io_wait()

首先，如果有连接超时时间的话，就设置socket fd为非阻塞的，然后调用标准的socket函数connect()来发起连接，这个函数会立即返回-1并且设置errno为 EINPROGRESS或者EALREADY。然后，调用vio_io_wait()来使用poll()来阻塞等待这个连接可以写入，使用连接超时值作为poll()的超时参数。这样，在等待超时后就认为连接失败，否则连接就成功了。这里并没有使用网络协议自己的连接超时，因为那样的话无法在不影响其他进程的情况下随时灵活更改这个超时时间。

## **读写超时**

相关函数：vio_read(), vio_write(), vio_socket_io_wait()

首先，调用标准的recv()/send()系统调用来读取或者写入，如果有读写超时时间的话，就使用MSG_DONTWAIT作为最后一个flags参数，这样如果没有数据可以读取或者无法写入（比如网络拥塞）这个函数会立刻返回SOCKET_EAGAIN 或者SOCKET_EWOULDBLOCK，于是，调用vio_socket_io_wait()来阻塞等待这个socket fd可以读或者写，这个函数主要是调用了poll()，并且用读写超时值作为poll()的等待超时时间。

## **会话超时**

这里的会话(Session)其实就是数据库连接，在mysql内部对应于THD类。会话超时是Mysql server端才有的机制，在客户端没有。实现方法是：Mysql server内部线程定期检查所有THD会话对象的状态，将会话超时的THD对象销毁。
