# http header -websocket

首先，解答本题， http 通过判断 header 中是否包含 `Connection: Upgrade` 与 `Upgrade: websocket` 来判断当前协议是否要升级到 websocket ，下面我们了解一下 WebSocket 协议与由来

## WebSocket 由来

WebSocket 之前，如果需要在客户端和服务之间双向通信，需要通过 HTTP 轮询来实现， HTTP 轮询分为轮询与长轮询：

![图片](https://mmbiz.qpic.cn/mmbiz/bwG40XYiaOKkQObgjMYF8aHkFSsl7YKbmlV3NCOmRAp8a30R1McMVlLRjooooMN7dkqnB3abfdVBvYcMDv3QKyg/640?wx_fmt=other&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

其中，轮询是指浏览器通过 JavaScript 启动一个定时器，然后以固定的间隔给服务器发请求，询问服务器有没有新消息，缺点：

- 实时性不够
- 频繁的请求会给服务器带来极大的压力

长轮询是指浏览器发送一个请求时，服务器先拖一段时间，等到有消息了再回复。这个机制暂时地解决了实时性问题，但是它带来了新的问题：

- 以多线程模式运行的服务器会让大部分线程大部分时间都处于挂起状态，极大地浪费服务器资源
- 一个HTTP连接在长时间没有数据传输的情况下，链路上的任何一个网关都可能关闭这个连接，而网关是我们不可控的

因此，HTML5 新增了 WebSocket 协议，能够在浏览器和服务器之间建立一个不受限的双向通信的通道。

> **为什么WebSocket连接可以实现全双工通信而HTTP连接不行呢？**
>
> 实际上HTTP协议是建立在TCP协议之上的，TCP协议本身就实现了全双工通信，但是HTTP协议的请求－应答机制限制了全双工通信。WebSocket连接建立以后，其实只是简单规定了一下：接下来，咱们通信就不使用HTTP协议了，直接互相发数据吧。

WebSocket 的优点：

- 较少的控制开销：在连接创建后，服务器和客户端之间交换数据时，用于协议控制的数据包头部相对较小
- 更强的实时性：由于协议是全双工的，所以服务器可以随时主动给客户端下发数据
- 保持连接状态：与 HTTP 不同的是，WebSocket 需要先创建连接，这就使得其成为一种有状态的协议，之后通信时可以省略部分状态信息
- 更好的二进制支持：WebSocket 定义了二进制帧，相对 HTTP，可以更轻松地处理二进制内容
- 可以支持扩展：WebSocket 定义了扩展，用户可以扩展协议、实现部分自定义的子协议

## WebSocket 协议

WebSocket 使用 `ws` 或 `wss` 的统一资源标志符（URI），其中 `wss` 表示使用了 TLS 的 WebSocket。

> `ws://` 数据不是加密的，对于任何中间人来说其数据都是可见的。
>
> `wss://` 是基于 TLS 的 WebSocket，类似于 HTTPS 是基于 TLS 的 HTTP），传输安全层在发送方对数据进行了加密，在接收方进行解密

http 通过判断 header 中是否包含 `Connection: Upgrade` 与 `Upgrade: websocket` 来判断当前是否需要升级到 websocket 协议，除此之外，还有其它 header：

![图片](https://mmbiz.qpic.cn/mmbiz_png/bwG40XYiaOKkQObgjMYF8aHkFSsl7YKbmRnp5mTPglK8Eqk7saa4YtTHpM27P8BkwsD6PMFO0fZWWRDaUOFUgrg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

- `Sec-WebSocket-Key` ：浏览器随机生成的安全密钥
- `Sec-WebSocket-Version` ：WebSocket 协议版本
- `Sec-WebSocket-Extensions` ：用于协商本次连接要使用的 WebSocket 扩展
- `Sec-WebSocket-Protocol` ：协议

当服务器同意进行 WebSocket 连接时，返回响应码 `101`

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

测试地址：https://www.websocket.org/echo.html

一旦 socket 被建立，我们就应该监听 socket 上的事件。一共有 4 个事件：

- **open** ：连接已建立
- **message** ：接收到数据
- **error** ：WebSocket 错误
- **close** ：连接已关闭

如果我们想发送消息，可以使用 `socket.send(data)`

```
let socket = new WebSocket("wss://echo.websocket.org")

socket.onopen = function(e) {
  console.log("[open] Connection established")
  // 发送消息
  socket.send("My name is an")
}

socket.onmessage = function(event) {
  console.log(`[message] Data received from server: ${event.data}`)
}

socket.onclose = function(event) {
  // ...
}

socket.onerror = function(error) {
  console.log(`[error] ${error.message}`)
}
```

## 总结

WebSocket 使用 `ws` 或 `wss` 的统一资源标志符，通过判断 header 中是否包含 `Connection: Upgrade` 与 `Upgrade: websocket` 来判断当前是否需要升级到 websocket 协议，除此之外，它还包含 `Sec-WebSocket-Key` 、 `Sec-WebSocket-Version` 等header，当服务器同意 WebSocket 连接时，返回响应码 `101` ，它的 API 很简单。

方法：

- `socket.send(data)`
- `socket.close([code], [reason])`

事件：

- `open`
- `message`
- `error`
- `close`

