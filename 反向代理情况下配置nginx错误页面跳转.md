如果是反向代理，后台Tomcat 处理报错抛出404，想把这个状态让Nginx反馈给客户端或者重定向到某个连接。

在 server/location 内开启以下变量，我们才能自定义错误页面

proxy_intercept_errors on;
