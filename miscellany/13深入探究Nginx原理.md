title: 深入Nginx原理
tag: nginx
---

Nginx是一个高性能的HTTP和反向代理服务器，及电子邮件（IMAP/POP3）代理服务器，同时也是一个非常高效的反向代理、负载平衡中间件。是非常常用的web server.我们需要理解它的原理，才能达到游刃有余的程度。
<!-- more -->

本篇文章需要对Nginx有基本的使用以及对IO复用模型有一定的了解。文章比较长。


## 1.正向代理和反向代理


![image](http://xiaozhao.oursnail.cn/%E6%AD%A3%E5%90%91%E4%BB%A3%E7%90%86.png)

**正向代理的工作原理就像一个跳板**，比如：我访问不了`google.com`，但是我能访问一个代理服务器A，A能访问`google.com`，于是我先连上代理服务器A，告诉他我需要`google.com`的内容，A就去取回来，然后返回给我。从网站的角度，只在代理服务器来取内容的时候有一次记录，有时候并不知道是用户的请求，也隐藏了用户的资料，这取决于代理告不告诉网站。

![image](http://xiaozhao.oursnail.cn/%E5%8F%8D%E5%90%91%E4%BB%A3%E7%90%86.png)

反向代理（`Reverse Proxy`）方式是指以代理服务器来接受`internet`上的连接请求，然后将请求转发给内部网络上的服务器，并将从服务器上得到的结果返回给`internet`上请求连接的客户端，此时代理服务器对外就表现为一个服务器。


简单来说：
- 正向代理是不知道客户端是谁，代理是一个跳板，所有客户端通过这个跳板来访问到对应的内容。
- 反向代理是不知道服务端是谁，用户的请求被转发到内部的某台服务器去处理。

## 2.基本的工作流程

![image](http://xiaozhao.oursnail.cn/nginx%E5%9F%BA%E6%9C%AC%E5%B7%A5%E4%BD%9C%E6%B5%81%E7%A8%8B.jpg)

1. 用户通过域名发出访问Web服务器的请求，该域名被DNS服务器解析为反向代理服务器的IP地址；
2. 反向代理服务器接受用户的请求；
3. 反向代理服务器在本地缓存中查找请求的内容，找到后直接把内容发送给用户；
4. 如果本地缓存里没有用户所请求的信息内容，反向代理服务器会代替用户向源服务器请求同样的信息内容，并把信息内容发给用户，如果信息内容是缓存的还会把它保存到缓存中。

## 3.优点

- 保护了真实的web服务器，保证了web服务器的资源安全
- 节约了有限的IP地址资源
- 减少WEB服务器压力，提高响应速度(缓存功能)
- 请求的统一控制，包括设置权限、过滤规则等
- 实现负载均衡
- 区分动态和静态可缓存内容
- ......



## 4.使用场景

- Nginx作为Http代理、反向代理
- Nginx作为负载均衡器
- Ip hash算法，对客户端请求的ip进行hash操作，然后根据hash结果将同一个客户端ip的请求分发给同一台服务器进行处理，可以解决session不共享的问题。
- Nginx作为Web缓存



## 5.Nginx的Master-Worker模式

![image](http://xiaozhao.oursnail.cn/Nginx%E7%9A%84Master-Worker%E6%A8%A1%E5%BC%8F.png)

启动`Nginx`后，其实就是在80端口启动了`Socket`服务进行监听，如图所示，`Nginx`涉及`Master`进程和`Worker`进程。

![image](http://xiaozhao.oursnail.cn/Master-Worker%E6%A8%A1%E5%BC%8F.png)

## 6.Master进程的作用是？

读取并验证配置文件`nginx.conf`；管理`worker`进程；

- 接收来自外界的信号
- 向各worker进程发送信号
- 监控worker进程的运行状态，当worker进程退出后(异常情况下)，会自动重新启动新的worker进程

## 7.Worker进程的作用是？

每一个`Worker`进程都维护一个线程（避免线程切换），处理连接和请求；注意`Worker`进程的个数由配置文件决定，一般和CPU个数相关（有利于进程切换），配置几个就有几个`Worker`进程。


##### 思考：Nginx如何做到热部署？

> 所谓热部署，就是配置文件nginx.conf修改后，不需要stop Nginx，不需要中断请求，就能让配置文件生效！（nginx -s reload 重新加载/nginx -t检查配置/nginx -s stop）
> 
> 通过上文我们已经知道worker进程负责处理具体的请求，那么如果想达到热部署的效果，可以想象：
> 
> 方案一：
> 
> 修改配置文件nginx.conf后，主进程master负责推送给woker进程更新配置信息，woker进程收到信息后，更新进程内部的线程信息。（有点volatile的味道）
> 
> 方案二：
> 
> 修改配置文件nginx.conf后，重新生成新的worker进程，当然会以新的配置进行处理请求，而且新的请求必须都交给新的worker进程，至于老的worker进程，等把那些以前的请求处理完毕后，kill掉即可。

Nginx采用的就是方案二来达到热部署的！


##### 思考：Nginx如何做到高并发下的高效处理？

> 上文已经提及Nginx的worker进程个数与CPU绑定、worker进程内部包含一个线程高效回环处理请求，这的确有助于效率，但这是不够的。
> 
> 作为专业的程序员，我们可以开一下脑洞：BIO/NIO/AIO、异步/同步、阻塞/非阻塞...
> 
> 要同时处理那么多的请求，要知道，有的请求需要发生IO，可能需要很长时间，如果等着它，就会拖慢worker的处理速度。
> 
> Nginx采用了Linux的epoll模型，epoll模型基于事件驱动机制，它可以监控多个事件是否准备完毕，如果OK，那么放入epoll队列中，这个过程是异步的。worker只需要从epoll队列循环处理即可。

##### 思考：Nginx挂了怎么办？

> Nginx既然作为入口网关，很重要，如果出现单点问题，显然是不可接受的。
> 
> 答案是：Keepalived+Nginx实现高可用。
> 
> Keepalived是一个高可用解决方案，主要是用来防止服务器单点发生故障，可以通过和Nginx配合来实现Web服务的高可用。（其实，Keepalived不仅仅可以和Nginx配合，还可以和很多其他服务配合）
> 
> Keepalived+Nginx实现高可用的思路：
> 
> 第一：请求不要直接打到Nginx上，应该先通过Keepalived（这就是所谓虚拟IP，VIP）
> 
> 第二：Keepalived应该能监控Nginx的生命状态（提供一个用户自定义的脚本，定期检查Nginx进程状态，进行权重变化,从而实现Nginx故障切换）

![image](http://xiaozhao.oursnail.cn/Keepalived+Nginx.png)

## 6.nginx.conf

![image](http://xiaozhao.oursnail.cn/nginx.conf.png)

- 第一：location可以进行正则匹配，应该注意正则的几种形式以及优先级。（这里不展开）
- 第二：Nginx能够提高速度的其中一个特性就是：动静分离，就是把静态资源放到Nginx上，由Nginx管理，动态请求转发给后端。
- 第三：我们可以在Nginx下把静态资源、日志文件归属到不同域名下（也即是目录），这样方便管理维护。
- 第四：Nginx可以进行IP访问控制，有些电商平台，就可以在Nginx这一层，做一下处理，内置一个黑名单模块，那么就不必等请求通过Nginx达到后端在进行拦截，而是直接在Nginx这一层就处理掉。

除了可以映射静态资源，上面已经说了，可以作为一个代理服务器来使用。


> 所谓反向代理，很简单，其实就是在location这一段配置中的root替换成proxy_pass即可。root说明是静态资源，可以由Nginx进行返回；而proxy_pass说明是动态请求，需要进行转发，比如代理到Tomcat上。
> 
> 反向代理，上面已经说了，过程是透明的，比如说request -> Nginx -> Tomcat，那么对于Tomcat而言，请求的IP地址就是Nginx的地址，而非真实的request地址，这一点需要注意。不过好在Nginx不仅仅可以反向代理请求，还可以由用户自定义设置HTTP HEADER。


负载均衡【upstream】

> 上面的反向代理中，我们通过proxy_pass来指定Tomcat的地址，很显然我们只能指定一台Tomcat地址，那么我们如果想指定多台来达到负载均衡呢？
> 
> 第一，通过upstream来定义一组Tomcat，并指定负载策略（IPHASH、加权论调、最少连接），健康检查策略（Nginx可以监控这一组Tomcat的状态）等。
> 
> 第二，将proxy_pass替换成upstream指定的值即可。
> 
> 负载均衡可能带来的问题？
> 
> 负载均衡所带来的明显的问题是，一个请求，可以到A server，也可以到B server，这完全不受我们的控制，当然这也不是什么问题，只是我们得注意的是：用户状态的保存问题，如Session会话信息，不能在保存到服务器上。

## 7.惊群现象

定义：惊群效应就是当一个fd的事件被触发时，所有等待这个fd的线程或进程都被唤醒。

`Nginx`的IO通常使用`epoll`，`epoll`函数使用了I/O复用模型。与I/O阻塞模型比较，I/O复用模型的优势在于可以同时等待多个（而不只是一个）套接字描述符就绪。`Nginx`的`epoll`工作流程如下：

- master进程先建好需要listen的socket后，然后再fork出多个woker进程，这样每个work进程都可以去accept这个socket
- 当一个client连接到来时，所有accept的work进程都会受到通知，但只有一个进程可以accept成功，其它的则会accept失败，Nginx提供了一把**共享锁accept_mutex**来保证同一时刻只有一个work进程在accept连接，从而解决惊群问题
- 当一个worker进程accept这个连接后，就开始读取请求，解析请求，处理请求，产生数据后，再返回给客户端，最后才断开连接，这样一个完成的请求就结束了


## 8.Nginx架构及工作流程

![image](http://xiaozhao.oursnail.cn/Nginx%E6%9E%B6%E6%9E%84%E5%8F%8A%E5%B7%A5%E4%BD%9C%E6%B5%81%E7%A8%8B%E5%9B%BE.png)

`Nginx`真正处理请求业务的是`Worker`之下的线程。`worker`进程中有一个`ngx_worker_process_cycle()`函数，执行无限循环，不断处理收到的来自客户端的请求，并进行处理，直到整个`Nginx`服务被停止。

当一个 `worker` 进程在 `accept()` 这个连接之后，就开始读取请求，解析请求，处理请求，产生数据后，再返回给客户端，最后才断开连接，一个完整的请求。一个请求，完全由 `worker` 进程来处理，而且只能在一个 `worker` 进程中处理。

这样做带来的好处：

1. 节省锁带来的开销。每个 `worker` 进程都是独立的进程，不共享资源，不需要加锁。同时在编程以及问题查上时，也会方便很多。
2. 独立进程，减少风险。采用独立的进程，可以让互相之间不会影响，一个进程退出后，其它进程还在工作，服务不会中断，`master` 进程则很快重新启动新的 `worker` 进程。当然，`worker` 进程的也能发生意外退出。

## 9.nginx为什么高性能

**因为nginx是多进程单线程的代表，多进程模型每个进程/线程只能处理一路IO，那么 Nginx是如何处理多路IO呢？**

如果不使用 IO 多路复用，那么在一个进程中，同时只能处理一个请求，比如执行 `accept()`，如果没有连接过来，那么程序会阻塞在这里，直到有一个连接过来，才能继续向下执行。

而多路复用，允许我们只在事件发生时才将控制返回给程序，而其他时候内核都挂起进程，随时待命。

**核心：Nginx采用的 IO多路复用模型epoll**

`epoll`通过在`Linux`内核中申请一个简易的文件系统（文件系统一般用什么数据结构实现？B+树），其工作流程分为三部分：


1. 调用 `int epoll_create(int size)`建立一个`epoll`对象，内核会创建一个`eventpoll`结构体，用于存放通过`epoll_ctl()`向`epoll`对象中添加进来的事件，这些事件都会挂载在红黑树中。
2. 调用 `int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event)` 在 `epoll` 对象中为 fd 注册事件，所有添加到`epoll`中的事件都会与设备驱动程序建立回调关系，也就是说，当相应的事件发生时会调用这个`sockfd`的回调方法，将`sockfd`添加到`eventpoll` 中的双链表
3. 调用 `int epoll_wait(int epfd, struct epoll_event * events, int maxevents, int timeout)` 来等待事件的发生，`timeout` 为 -1 时，该调用会阻塞直到有事件发生

这样，注册好事件之后，只要有 fd 上事件发生，`epoll_wait()` 就能检测到并返回给用户，用户就能”非阻塞“地进行 I/O 了。

`epoll()` 中内核则维护一个链表，`epoll_wait` 直接检查链表是不是空就知道是否有文件描述符准备好了。（`epoll` 与 `select` 相比最大的优点是不会随着 `sockfd` 数目增长而降低效率，使用 `select()` 时，内核采用轮训的方法来查看是否有fd 准备好，其中的保存 `sockfd` 的是类似数组的数据结构 `fd_set`，key 为 fd，value 为 0 或者 1。）

能达到这种效果，是因为在内核实现中 `epoll` 是根据每个 `sockfd` 上面的与设备驱动程序建立起来的回调函数实现的。那么，某个 `sockfd` 上的事件发生时，与它对应的回调函数就会被调用，来把这个 `sockfd` 加入链表，其他处于“空闲的”状态的则不会。在这点上，`epoll` 实现了一个”伪”AIO。但是如果绝大部分的 I/O 都是“活跃的”，每个 `socket` 使用率很高的话，`epoll`效率不一定比 `select` 高（可能是要维护队列复杂）。

可以看出，因为一个进程里只有一个线程，所以一个进程同时只能做一件事，但是可以通过不断地切换来“同时”处理多个请求。

例子：`Nginx` 会注册一个事件：“如果来自一个新客户端的连接请求到来了，再通知我”，此后只有连接请求到来，服务器才会执行 `accept()` 来接收请求。又比如向上游服务器（比如 PHP-FPM）转发请求，并等待请求返回时，这个处理的 `worker` 不会在这阻塞，它会在发送完请求后，注册一个事件：“如果缓冲区接收到数据了，告诉我一声，我再将它读进来”，于是进程就空闲下来等待事件发生。

这样，基于 多进程+epoll， Nginx 便能实现高并发。


## 10.几种负载均衡的算法介绍

> 轮询（默认）

每个请求按时间顺序逐一分配到不同的后端服务器，如果后端服务器down掉，能自动剔除。

> weight

指定轮询几率，weight和访问比率成正比，用于后端服务器性能不均的情况。

> ip_hash

每个请求按访问ip的hash结果分配，这样每个访客固定访问同一个后端服务器，可以解决session的问题。但是不能解决宕机问题。
前三种是nginx自带的，直接在配置文件中配置即可使用。

> fair（第三方）

按后端服务器的相应时间来分配请求，相应时间短的优先分配。

> url_hash（第三方）

按访问url的hash结果来分配请求，使每个url定向到同一个后端服务器，后端服务器为缓存时比较有效。


## 11.基于不同层次的负载均衡

![image](http://bloghello.oursnail.cn/http1-6.png)

七层就是基于URL等应用层信息的负载均衡；
同理，还有基于MAC地址的二层负载均衡和基于IP地址的三层负载均衡。

换句话说:
- 二层负载均衡会通过一个虚拟MAC地址接受请求，然后再分配到真是的MAC地址；
- 三层负载均衡会通过一个虚拟IP地址接收请求，然后再分配到真实的IP地址；
- 四层通过虚拟的URL或主机名接收请求，然后再分配到真是的服务器。

所谓的四到七层负载均衡，就是在对后台的服务器进行负载均衡时，依据四层的信息或七层的信息来决定怎么样转发流量。

比如四层的负载均衡，就是通过发布三层的IP地址（VIP），然后加四层的端口号，来决定哪些流量需要做负载均衡，对需要处理的流量进行NAT处理，转发至后台服务器，并记录下这个TCP或者UDP的流量是由哪台服务器处理的，后续这个连接的所有流量都同样转发到同一台服务器处理。

七层的负载均衡，就是在四层的基础上（没有四层是绝对不可能有七层的），再考虑应用层的特征，比如同一个Web服务器的负载均衡，除了根据VIP加80端口辨别是否需要处理的流量，还可根据七层的URL、浏览器类别、语言来决定是否要进行负载均衡。举个例子，如果你的Web服务器分成两组，一组是中文语言的，一组是英文语言的，那么七层负载均衡就可以当用户来访问你的域名时，自动辨别用户语言，然后选择对应的语言服务器组进行负载均衡处理。

负载均衡器通常称为四层交换机或七层交换机。四层交换机主要分析IP层及TCP/UDP层，实现四层流量负载均衡。七层交换机除了支持四层负载均衡以外，还有分析应用层的信息，如HTTP协议URI或Cookie信息。

负载均衡设备也常被称为"四到七层交换机"，那么四层和七层两者到底区别在哪里？

> 第一，技术原理上的区别。

所谓四层负载均衡，也就是主要通过报文中的目标地址和端口，再加上负载均衡设备设置的服务器选择方式，决定最终选择的内部服务器。

所谓七层负载均衡，也称为“内容交换”，也就是主要通过报文中的真正有意义的应用层内容，再加上负载均衡设备设置的服务器选择方式，决定最终选择的内部服务器。

> 第二，应用场景的需求。

七层应用负载的好处，是使得整个网络更"智能化"。例如访问一个网站的用户流量，可以通过七层的方式，将对图片类的请求转发到特定的图片服务器并可以使用缓存技术；将对文字类的请求可以转发到特定的文字服务器并可以使用压缩技术。

另外一个常常被提到功能就是安全性。

## 12.总结

1. 理解正向代理和反向代理的概念
2. nginx的优点和使用场景
3. master和work两种进程的作用
4. 如何热部署
5. Nginx单点故障的预防
6. 映射静态文件、反向代理跳转到后端服务器处理的写法
7. 惊群现象
8. Nginx 采用的是多进程（单线程） & 多路IO复用模型(底层依靠epoll实现)
9. 几种负载均衡的算法
10. 四层的负载均衡和七层的负载均衡




