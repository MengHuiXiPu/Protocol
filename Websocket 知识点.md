# Websocket 知识点

# 一、首先我们要了解 Websocket 握手的原理

![图片](https://mmbiz.qpic.cn/mmbiz_png/ndgH50E7pIpoC4GCCpgaSlCwdbzUhSsKqiaHrg5DsFPGAwNklcR0vj8eUBN1UyVMsB0SHd6frF2QtDmjCUfEJQA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

## 请求头特征

- HTTP 必须是 1.1 GET 请求

- HTTP Header 中 Connection 字段的值必须为 Upgrade

- HTTP Header 中 Upgrade 字段必须为 websocket

- Sec-WebSocket-Key 字段的值是采用 base64 编码的随机 16 字节字符串

- Sec-WebSocket-Protocol 字段的值记录使用的子协议，比如 binary base64

- Origin 表示请求来源

## 响应头特征

- 状态码是 101 表示 Switching Protocols

- Upgrade / Connection / Sec-WebSocket-Protocol 和请求头一致

- Sec-WebSocket-Accept 是通过请求头的 Sec-WebSocket-Key 生成

# 二、短连接轮询、长连接、Websocket 横向对比

## 1. 短连接轮询

- 很耗费 TCP 连接

- 而且 Header 重复发送

- 且通过宏任务发起，受限于 Event Loop，无法保证及时性

- 同时无效请求会很多

## 2. 长连接

- HTTP keep-alive 开启后虽然 TCP 可以复用，但是 Header 重复的问题并没有解决

- 同时 HTTP keep-alive 还有一个有效期，有效期结束后服务端会发侦查帧探查 TCP 是否有效

> 题外话：

> **HTTP keep-alive 的作用是，告知服务端持久化当前的** **TCP** **连接，不要立即断开，以便后续的 HTTP 请求复用它，也就是我们所说的「长连接」**

> HTTP 的 keep-alive 是为了让 TCP 活久一点，而 TCP 本身也有一个 keepalive（注意没有横杠哦）机制。这是 TCP 的一种检测连接状况的保活机制，keepalive 是 TCP 保活定时器：TCP 建立后，如果闲置没用，服务器不可能白等下去，闲置一段时间[可设置]后，服务器就会尝试向客户端发送侦测包，来判断 TCP 连接状况，如果没有收到对方的回答（ACK包），就会过一会[可设置]再侦测一次，如果多次[可设置]都没回答，就会丢弃这个 TCP 连接

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

（TCP keepalive 保活示意图）

## 3. Websocket

- 和 HTTP 一样都是建立在 TCP 协议之上，但只需一次 HTTP 握手，就能建立持久性连接，后续就不走 HTTP 了,而是 WebSocket 特有的数据帧

- 全双工通信，双向数据传输

- 数据格式轻量，且支持发送二进制数据，支持 ws 和加密的 wss

# 三、我在微信小程序中利用 WebSocket 都捣鼓了什么？

## 1 验签鉴权及对应的容错策略（登录态要求、峰值访问、服务端宕机异常）

背景与目的：

- websocket 握手后，接口请求即可以放弃 HTTP 改走 weboskcet，但大部分业务接口都要求登录态，因此握手成功后必须先走一次签名鉴权，获取登录态

- 当出现大流量访问的场景（如大促、热点活动等）或服务端出 bug 而导致服务端宕机，前端会做 对应容错，将位于内存的等待队列中的待发送请求立即降级成 HTTP 发送出去

伪码示意：

```
SocketTask.onOpen(function () {
  SocketTask.sendSocketMessage({
     msg_type: '验签'，
     token: 'xxx'
  }, (response) => {
      console.log(response.user_id, response.access_token)

      // 通道可用，打个标记
      global.isSocketAvaliable = true;
  })
})
```

## 2 心跳保活（减少 TCP 占用）

背景与目的：为了减少 TCP 连接的无效占用，客户端定时发送一个空包到服务端，告知服务端不要销毁这条 socket，如果服务端超过一定时间都没收到心跳包，则将关闭并销毁该 socket

伪码示意：

```
SocketTask.onOpen(function () {
  SocketTask.sendSocketMessage({
     msg_type: '验签'，
     token: 'xxx'
  }, (response) => {
      console.log(response.user_id, response.access_token)

      // 通道可用，打个标记
      global.isSocketAvaliable = true;
      
      // 验签成功，开始定时发送心跳包
      setInterval(() => {
          SocketTask.sendSocketMessage({
            msg_type: '心跳'
          });
      });
   });
})
```

## 3 模拟 RTT（用于弱网体验优化）

背景与目的：在发送心跳包时，可得知一个心跳包的 RTT，以此模拟当前用户网络环境的 TCP RTT，并据此计算出平滑 RTO，用于弱网体验优化

伪码示意：

```
SocketTask.onOpen(function () {
  SocketTask.sendSocketMessage({
     msg_type: '验签'，
     token: 'xxx'
  }, (response) => {
      console.log(response.user_id, response.access_token)

      // 通道可用，打个标记
      global.isSocketAvaliable = true;
      
      // 验签成功，开始定时发送心跳包
      setInterval(() => {
          // 计算 RTT
          const begin = Date.now();

          SocketTask.sendSocketMessage({
            msg_type: '心跳'
          }, () => {
            const end = Date.now();
            
            const RTT = begin - end;
            
            const smoothedRTO = cal(RTT);
            
            global.smoothedRTO = smoothedRTO;
          });
      });
   });
});
```

## 4 Snappy 压缩（横向对比了 gzip / zip / 7z）

背景与目的：在小程序中引入第三方压缩包（牺牲小程序包体积），减少 websocket 传输的字节数

伪码示意：

```
  import Snappy from 'snappy';

  SocketTask.sendSocketMessage = function (msg) {
     const encryptedMsg = Snappy.encode(msg);
     
     wx.send(encryptedMsg);
  }
```

## 5 重连（阶梯式错位重连，避免拥挤）

背景与目的：用户的网络环境不稳定，可能会存在主动 / 被动断开 socket 的情况，需要进行自动重连

伪码示意：

```
SocketTask.onClose(function () {
  // 限定最大重连次数
  if (retryCount > maxCount) {
    return;
  }
  
  retryCount++;

  setTimeout(() => {
    SocketTask.connectSocket();
  }, retryCount * 1000 + Math.random() * 1000);
});
```

## 6 埋点中间层缓存（重复的用户信息可以不用每次都上报，支持刷新缓存）

背景与目的：为减少网络传输的包体积，通过 websocket 上报埋点日志时，可以把部分重复字段值在第一次上报时缓存在服务端，从第二次上报开始只上报值不重复的字段，然后由服务端做日志合并

伪码示意：

```
SocketTask.sendSocketMessage({
     msg_type: '埋点日志'，
     logs: {
       country: 'China', // 可缓存字段
       city: '北京', // 可缓存字段
       platform: '安卓', // 可缓存字段
       click_some_btn: true // 动态变化的埋点字段
     },
     cacheFields: ['country', 'city', 'platform'] // 只在第一次上报时携带
 });
```

## 7 启用 TCP_NODELAY

TCP_NODELAY 是用来禁用 Nagle 算法的。Nagle 算法设计的目的是提高网络带宽利用率，其核心思路是「合并小的 TCP 包为一个大的 TCP 包」，避免过多的小包的 TCP 头部浪费网络带宽