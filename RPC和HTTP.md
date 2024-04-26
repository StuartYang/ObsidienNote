netty，dubbo，grpc，open ferign



# RPC和HTTP

HTTP 它是协议，不是运输通道，传输的任务由于 TCP 建立握手，然后进行数据传递。那 HTTP 用来干什么呢，这就涉及到浏览器和服务器之间的通信交互。HTTP 协议定义了，浏览器通过 HTTP 协议所规定的格式进行加载服务穿来的数据，按照格式要求放在页面上。例如 HTTP 有请求头，响应体等。

 RPC 则是远程调用，其对应的是本地调用。RPC 的通信可以用 HTTP 协议，也可以自定义协议，是不做约束的。

```java
public User getUserById(Long id) {
       return userDao.getUserById(id); // 这叫本地调用
}
```

现在都是分布式架构，根据业务模块做了不同的拆分，不同的模块进行通信就需要远程调用。涉及到交互，就需要拟定一个通信协议，至于用不用 HTTP 都可以。由于 HTTP 协议比较冗余，自己内部使用不需要那么麻烦，不需要考虑通用性，只需要项目内保持一致即可，所以可以定制化协议来进行通信。



所以 RPC 大多数都是基于 TCP/IP 协议实现的定制化协议。



![未命名绘图.drawio](https://cdn.jsdelivr.net/gh/StuartYang/oss@master/img/202401130153026.png)



- XXX stub：客户端存根负责对信息进行序列化和反序列化然后放在 TCP 传输通道上进行传输。



**同步调用与异步调用**

- 同步调用：客户端等待调用执行完成并返回结果。

- 异步调用：客户端不等待调用执行完成返回结果，不过依然可以通过回调函数等接收到返回结果的通知。如果客户端并不关心结果，则可以变成一个单向的调用。

HTTP协议：是一个有来有回的协议。

RPC 协议：可以定制为异步协议或者同步协议。



# 流行的 RPC 框架

- **gRPC** 是 Google 最近公布的开源软件，基于最新的 HTTP2.0 协议，并支持常见的众多编程语言。
- **Thrift** 是 Facebook 的一个开源项目，主要是一个跨语言的服务开发框架。
- **Dubbo** 是阿里集团开源的一个极为出名的 RPC 框架，默认Dubbo协议，下层 TCP 协议，NIO异步传输。



# Dubbo和OpenFeign怎么选择

zk+dubbo的搭配多一些，但是springcloud项目中大多使用openfeign。

- Dubbo默认的Dubbo协议，下层基于TCP进行数据传输，传输会更加高效。
- openfeign基于HTTP进行数据传输，属于应用层。
- Dubbo虽然支持多种协议，但是需要显示的定义接口和实现类，配置各种参数。
- openfeign则更加关注于RESTful风格的接口调用，更加简单容易操作。