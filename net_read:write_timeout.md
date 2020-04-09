net_write/read_timeout

sql/sql_connect.cc

`bool login_connection(THD *thd)
{
  NET *net= &thd->net;
  int error= 0;
  DBUG_ENTER("login_connection");
  DBUG_PRINT("info", ("login_connection called by thread %lu",
                      (ulong) thd->thread_id));`

`  /* Use "connect_timeout" value during connection phase */
  my_net_set_read_timeout(net, connect_timeout);
  my_net_set_write_timeout(net, connect_timeout);`

`  ...`

`   /* Connect completed, set read/write timeouts back to default */
  my_net_set_read_timeout(net, thd->variables.net_read_timeout);
  my_net_set_write_timeout(net, thd->variables.net_write_timeout);`

-------------------------------------------

建立连接阶段：connection_timeoiut

server等待请求：wait_timeout(交互模式)，interactive_timeout(非交互模式)

请求进行期间：net_read_timeout, net_write_timeout
