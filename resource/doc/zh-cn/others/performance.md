# webman性能


###传统框架请求处理流程

1. nginx/apache接收请求
2. nginx/apache将请求传递给php-fpm
3. php-fpm初始化环境，如创建变量列表
4. php-fpm调用各个扩展/模块的RINIT
5. php-fpm磁盘读取php文件(使用opcache可避免)
6. php-fpm词法分析、语法分析、编译成opcode(使用opcache可避免)
7. php-fpm执行opcode 包括8.9.10.11
8. 框架初始化，如实例化各种类，包括如容器、控制器、路由、中间件等。
9. 框架连接数据库并权限验证，连接redis
10. 框架执行业务逻辑
11. 框架关闭数据库、redis连接
12. php-fpm释放资源、销毁所有类定义、实例、销毁符号表等
13. php-fpm顺序调用各个扩展/模块的RSHUTDOWN方法
14. php-fpm将结果转发给nginx/apache
15. nginx/apache将结果返回给客户端


###webman的请求处理流程
1. 框架接收请求
2. 框架执行业务逻辑
3. 框架将结果返回给客户端

没错，在没有nginx反代的情况下，框架只有这3步。可以说这已经是php框架的极致，这使得webman性能是传统框架的几倍甚至数十倍。

性能对比参考 [techempower.com 第20轮压测(带数据库业务)](https://www.techempower.com/benchmarks/#section=data-r20&hw=ph&test=db&l=zik073-sf&a=2)

### 压力测试说明
#### 开启keep-alive
目前浏览器都是默认开启keep-alive的，也就是浏览器访问某一个http地址后会将连接暂时保留不关闭，下一次请求时复用这个连接，用来提高性能。
所以压测时建议开启keep-alive(如果是用的ab程序压测需要加-k参数，其它压测程序一般会默认开启)。
如果不开启keep-alive，每个请求都需要进行TCP三次握手和四次挥手，那么瓶颈会出现在操作系统内核网络处理部分，QPS会大幅下降。

#### 使用本机或者内网服务器压测
压力测试通过外网压测，性能会明显下降，这是因为压力机到服务器的网络传输成为瓶颈。
例如压力机到服务器网络延迟为50ms，那么100并发每秒最多只能发出去1000个http请求，这与语言以及框架无关。
如果没有开启keep-alive，那么QPS会更低，因为外网TCP三次握手和四次挥手比内网的占用时间会高出几个数量级。
外网压测要想能压出高吞吐量，必须提高并发，例如并发提高到10000并发。

#### 设置合适的进程数
webman默认开启cpu*4的进程数。实际上无网路IO的helloworld业务压测进程数开成与cpu核数一致性能最优，因为可以减少进程切换开销。
如果是带数据库、redis等阻塞IO业务，进程数可设置为cpu的3-8倍，因为这时需要更多的进程提高并发，而进程切换开销相对与阻塞IO则基本可以忽略。

#### 压测命令示例
`ab -n100000 -c200 -k http://127.0.0.1:8787/`
