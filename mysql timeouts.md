

 connect_timeout 指的是连接过程中握手的超时时间，默认为10秒。

mysql的基本原理是有个监听线程循环接收请求，当有请求来时，创建线程（或者从线程池中取）来处理这个请求。由于mysql连接采用TCP协议，那么之前势必是需要进行TCP三次握手的。TCP三次握手成功之后，客户端会进入阻塞，等待服务端的消息。服务端这个时候会创建一个线程(或者从线程池中取一个线程)来处理请求，主要验证部分包括host和用户名密码验证。host验证我们比较熟悉，因为在用grant命令授权用户的时候是有指定host的。用户名密码认证则是服务端先生成一个随机数发送给客户端，客户端用该随机数和密码进行多次sha1[加密](http://www.2cto.com/article/jiami/)后发送给服务端验证。如果通过，整个连接握手过程完成。（具体握手过程后续找到资料再分析）

由此可见，整个连接握手可能会有各种可能出错。所以这个connect_timeout值就是指这个超时时间了。



interactive_timeout & wait_timeout

从官方从文档上来看wait_timeout和interactive_timeout都是指不活跃的连接超时时间，连接线程启动的时候全局wait_timeout会根据是交互模式还是非交互模式被设置为这两个值中的一个。如果我们运行mysql -uroot -p命令登陆到mysql，wait_timeout就会被设置为interactive_timeout的值。如果我们在wait_timeout时间内没有进行任何操作，那么再次操作的时候就会提示超时，这时mysql client会重新连接。

On thread startup, the session [`wait_timeout`](https://docs.oracle.com/cd/E17952_01/mysql-5.7-en/server-system-variables.html#sysvar_wait_timeout) value is initialized from the global [`wait_timeout`](https://docs.oracle.com/cd/E17952_01/mysql-5.7-en/server-system-variables.html#sysvar_wait_timeout) value or from the global [`interactive_timeout`](https://docs.oracle.com/cd/E17952_01/mysql-5.7-en/server-system-variables.html#sysvar_interactive_timeout) value, depending on the type of client (as defined by the `CLIENT_INTERACTIVE` connect option to [`mysql_real_connect()`](https://docs.oracle.com/cd/E17952_01/mysql-5.7-en/mysql-real-connect.html "27.7.6.54 mysql_real_connect()")). See also [`interactive_timeout`](https://docs.oracle.com/cd/E17952_01/mysql-5.7-en/server-system-variables.html#sysvar_interactive_timeout).



net_read_timeout & net_write_timeout

文档中描述如下，就是说这两个参数在网络条件不好的情况下起作用。比如我在客户端用load data infile的方式导入很大的一个文件到[数据库](http://www.2cto.com/database/)中，然后中途用iptables禁用掉mysql的3306端口，这个时候服务器端该连接状态是reading from net，在等待net_read_timeout后关闭该连接。同理，在程序里面查询一个很大的表时，在查询过程中同样禁用掉端口，制造网络不通的情况，这样该连接状态是writing to net，然后在net_write_timeout后关闭该连接。

Usually `net_read_timeout` shouldn't be a problem but when you have some network trouble, especially when communicating with the server this timeout could be raised because instead of a single packet for the query, that you sent to the Database, MySQL waits for the entire query to be read but, due to the network problem, it doesn't receive the rest of the query. MySQL doesn't allow client to talk the server until the query result is fetched completely.

基于tcp的网络包传输，是有头有尾的。假设你现在要发送1000w条记录到客户端，发送过程中，网络有问题（断开或延迟），这时这个往网络上写数据的服务端进程（线程）就会等待net_write_timeout的时间，如果剩下的包没有继续发送过来，那就超时失败。
