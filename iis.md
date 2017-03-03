### 服务器的监听(IIS6.0+版本)     

1. 当请求到达服务器时，请求最终会到达TCPIP.SYS驱动程序，TCPIP.SYS将请求转发给HTTP.SYS网络驱动程序的请求队列中（可以理解为专门处理http请求的进程），当然在处理请求的过程中，HTTP.SYS进程会维护一个配置表用缓存请求的url和和应用程序池对应的关系。   

2. 当一个http请求被捕获到，HTTP.SYS会读取配置表，如果对应的应用程序没有启动，则HTTP.SYS会启动IIS相对应的应用程序。具体运行机制可以理解成为：   
 
 ![Http.SYS机制](/assets/HttpSys机制.png)


### IIS处理 