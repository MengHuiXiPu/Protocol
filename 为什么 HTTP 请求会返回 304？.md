# 为什么 HTTP 请求会返回 304？

 Koa 缓存的示例，为大家介绍 HTTP  **304** 状态码和 fresh 模块中的 `fresh` 函数是如何实现资源新鲜度检测的。如果你对浏览器的缓存机制还不了解的话

### 一、304 状态码

在 [HTTP 中的 ETag 是如何生成的？](https://mp.weixin.qq.com/s?__biz=MzI2MjcxNTQ0Nw==&mid=2247491652&idx=1&sn=6ece1636d4517a32bfb33845a5641f22&scene=21#wechat_redirect) 这篇文章中，介绍了 ETag 是如何生成的。在 ETag 实战环节，阿宝哥基于 `koa`、`koa-conditional-get`、`koa-etag` 和 `koa-static` 这些库，演示了在实际项目中如何利用 `ETag` 响应头和 `If-None-Match` 请求头实现资源的缓存控制。

```
// server.js
const Koa = require("koa");
const path = require("path");
const serve = require("koa-static");
const etag = require("koa-etag");
const conditional = require("koa-conditional-get");

const app = new Koa();

app.use(conditional()); // 使用条件请求中间件
app.use(etag()); // 使用etag中间件
app.use( // 使用静态资源中间件
  serve(path.join(__dirname, "/public"), {
    maxage: 10 * 1000, // 设置缓存存储的最大周期，单位为秒
  });
);

app.listen(3000, () => {
  console.log("app starting at port 3000");
});
```

在启动完服务器之后，我们打开 Chrome 开发者工具并切换到 Network 标签栏，然后在浏览器地址栏输入 `http://localhost:3000/` 地址，接着多次访问该地址（地址栏多次回车）。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/jQmwTIFl1V3smmqCwRiaxBVwRjNl6sLGAFNJ5VOEsQHiaE903HozXxA1Y6lxXMMkibdUyLl9mb1DLrEnmyyBIia3Cw/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

上图是阿宝哥多次访问的结果，在图中我们可以看到 200 和 304 状态码。其中 304 状态码表示资源在由请求头中的 `If-Modified-Since` 或 `If-None-Match` 参数指定的这一版本之后，未曾被修改。**在这种情况下，由于客户端仍然具有以前下载的副本，因此不需要重新传输资源。**

下面我们以 `index.js` 资源为例，来近距离观察一下 304 响应报文：

```
HTTP/1.1 304 Not Modified
Last-Modified: Sat, 29 May 2021 02:24:53 GMT
Cache-Control: max-age=10
ETag: W/"29-179b5f04654"
Date: Sat, 29 May 2021 02:25:26 GMT
Connection: keep-alive
```

对于以上的响应报文，在响应头中包含了 `Last-Modified`、`Cache-Control` 和 `ETag` 这些与缓存相关的字段请求 `index.js` 资源会返回 304 ？

| 状态码 |                             文章                             |
| :----: | :----------------------------------------------------------: |
|  101   | [你不知道的 WebSocket](https://mp.weixin.qq.com/s?__biz=MzI2MjcxNTQ0Nw==&mid=2247485740&idx=1&sn=a9243b90b709d0e138f7f5e5beaea9c0&scene=21#wechat_redirect) |
|  206   | [JavaScript 中如何实现大文件并行下载？](https://mp.weixin.qq.com/s?__biz=MzI2MjcxNTQ0Nw==&mid=2247490849&idx=1&sn=9d062c04baeb629d9b69a9fb4e7c3599&scene=21#wechat_redirect) |

### 二、为何返回 304 状态码

在前面的示例中，我们通过使用 `app.use` 方法注册了 3 个中间件：

```
app.use(conditional()); // 使用条件请求中间件
app.use(etag()); // 使用etag中间件
app.use( // 使用静态资源中间件
  serve(path.join(__dirname, "/public"), {
    maxage: 10 * 1000, // 设置缓存存储的最大周期，单位为秒
  })
);
```

首先注册的是 `koa-conditional-get` 中间件，该中间件用于处理 HTTP 条件请求。在这类请求中，请求的结果，甚至请求成功的状态，都会随着验证器与受影响资源的比较结果的变化而变化。**HTTP 条件请求可以用来验证缓存的有效性，省去不必要的控制手段。**

其实 `koa-conditional-get` 中间件的实现很简单，具体如下所示：

```
// https://github.com/koajs/conditional-get/blob/master/index.js
module.exports = function conditional () {
  return async function (ctx, next) {
    await next()
    if (ctx.fresh) {
      ctx.status = 304
      ctx.body = null
    }
  }
}
```

由以上代码可知，当请求上下文对象的 `fresh` 属性为 `true` 时，就会设置响应的状态码为 `304`。因此，接下来我们的重点就是分析 `ctx.fresh` 值的设置条件。

通过阅读 `koa/lib/context.js` 文件的源码，我们可知当访问上下文对象的 `fresh` 属性时，实际上是访问 `request` 对象的 `fresh` 属性。

```
// 代理request对象
delegate(proto, 'request')
   // 省略其它代理
  .getter('fresh')
  .getter('ips')
  .getter('ip');
```

而 request 对象上的 `fresh` 属性是通过 getter 方式来定义的，具体如下所示：

```
// node_modules/koa/lib/request.js
module.exports = {
  // 省略部分代码
  get fresh() {
    const method = this.method; // 获取请求方法
    const s = this.ctx.status; // 获取状态码

    if ('GET' !== method && 'HEAD' !== method) return false;

    // 2xx or 304 as per rfc2616 14.26
    if ((s >= 200 && s < 300) || 304 === s) {
      return fresh(this.header, this.response.header);
    }
    return false;
  },
}
```

在 `fresh` 方法中，仅当请求为 **GET/HEAD** 请求且状态码为 `2xx` 或 `304` 才会执行新鲜度检测。而对应的新鲜度检测逻辑被封装在 `fresh` 模块中，所以接下来我们来分析该模块是如何检测新鲜度？

### 三、如何检测新鲜度

fresh 模块对外提供了 `fresh` 函数，该函数支持 2 个参数：`reqHeaders` 和 `resHeaders`。在该函数内部，新鲜度检测的逻辑可以分为以下 4 个部分：

#### 3.1 判断是否条件请求

```
// https://github.com/jshttp/fresh/blob/master/index.js
function fresh (reqHeaders, resHeaders) {
  var modifiedSince = reqHeaders['if-modified-since'] 
  var noneMatch = reqHeaders['if-none-match']

  // 非条件请求
  if (!modifiedSince && !noneMatch) {
    return false
  }
}
```

如果请求头未包含 `if-modified-since` 和 `if-none-match` 字段，则直接返回 false。

#### 3.2 判断 cache-control 请求头

```
// https://github.com/jshttp/fresh/blob/master/index.js
var CACHE_CONTROL_NO_CACHE_REGEXP = /(?:^|,)\s*?no-cache\s*?(?:,|$)/

function fresh (reqHeaders, resHeaders) {
  var modifiedSince = reqHeaders['if-modified-since'] 
  var noneMatch = reqHeaders['if-none-match']
  
  // Always return stale when Cache-Control: no-cache
  // to support end-to-end reload requests
  // https://tools.ietf.org/html/rfc2616#section-14.9.4
  var cacheControl = reqHeaders['cache-control']
  if (cacheControl && CACHE_CONTROL_NO_CACHE_REGEXP.test(cacheControl)) {
    return false
  }
}
```

当 `cache-control` 请求头的值为 `no-cache` 时，则返回 false，以支持端到端的重载请求。**需要注意的是，`no-cache` 并不是表示不缓存，而是表示资源被缓存，但是立即失效，下次会发起请求验证资源是否过期。** 如果你不缓存任何响应，需要设置 `cache-control` 的值为 `no-store`。

#### 3.3 检测 ETag 是否匹配

```
// https://github.com/jshttp/fresh/blob/master/index.js
function fresh (reqHeaders, resHeaders) {
  var modifiedSince = reqHeaders['if-modified-since'] 
  var noneMatch = reqHeaders['if-none-match']
  
  // 省略部分代码
  if (noneMatch && noneMatch !== '*') {
    var etag = resHeaders['etag'] // 获取响应头中的etag字段的值

    if (!etag) { // 响应头未设置etag，则直接返回false
      return false
    }

    var etagStale = true // stale：不新鲜
    var matches = parseTokenList(noneMatch) // 解析noneMatch
    for (var i = 0; i < matches.length; i++) { // 执行循环匹配操作
      var match = matches[i]
      if (match === etag || match === 'W/' + etag || 'W/' + match === etag) {
        etagStale = false
        break
      }
    }

    if (etagStale) {
      return false
    }
  }
  return true
}
```

在以上代码中 `parseTokenList` 函数的作用，是为了处理 `'if-none-match': ' "bar" , "foo"'` 这种情形。在解析的过程中，会去掉多余的空格，并且还会拆分使用逗号分隔符做分隔的 `etag` 值。而执行循环匹配的目的，也是为了支持以下测试用例：

```
// https://github.com/jshttp/fresh/blob/master/test/fresh.js    
describe('when at least one matches', function () {
  it('should be fresh', function () {
    var reqHeaders = { 'if-none-match': ' "bar" , "foo"' }
    var resHeaders = { 'etag': '"foo"' }
    assert.ok(fresh(reqHeaders, resHeaders))
   })
})
```

此外，以上代码中的 `W/（大小写敏感）` 表示使用弱验证器。弱验证器很容易生成，但不利于比较。而如果 etag 中不包含 `W/`，则表示强验证器，它是比较理想的选择，但很难有效地生成。相同资源的两个弱 etag 值可能语义等同，但不是每个字节都相同。

#### 3.4 判断 Last-Modified 是否过期

```
// https://github.com/jshttp/fresh/blob/master/index.js
function fresh (reqHeaders, resHeaders) {
  var modifiedSince = reqHeaders['if-modified-since'] // 获取请求头中的修改时间
  var noneMatch = reqHeaders['if-none-match']

  // if-modified-since
  if (modifiedSince) {
    var lastModified = resHeaders['last-modified'] // 获取响应头中的修改时间
    var modifiedStale = !lastModified || !(parseHttpDate(lastModified) <= parseHttpDate(modifiedSince))

    if (modifiedStale) {
      return false
    }
  }

  return true
}
```

Last-Modified 的判断逻辑很简单，当响应头未设置 `last-modified` 字段信息或者响应头中 `last-modified` 的值大于请求头 `if-modified-since` 字段对应的修改时间时，则新鲜度的检测结果为 `false`，即表示资源已被修改过，已经不新鲜了。

了解完 `fresh` 函数的具体实现之后，我们再来回顾一下 `Last-Modified` 和 `ETag` 之间的区别：

- 精确度上，Etag 要优于 Last-Modified。Last-Modified 的时间单位是秒，如果某个文件在 1 秒内被改变多次，那么它们的 Last-Modified 并没有体现出来修改，但是 Etag 每次都会改变，从而确保了精度；此外，如果是负载均衡的服务器，各个服务器生成的 Last-Modified 也有可能不一致。
- 性能上，Etag 要逊于 Last-Modified，毕竟 Last-Modified 只需要记录时间，而 ETag 需要服务器通过消息摘要算法来计算出一个hash 值。
- **优先级上，在资源新鲜度校验时，服务器会优先考虑 Etag。** 即如果条件请求的请求头同时携带 `If-Modified-Since` 和 `If-None-Match` 字段，则会优先判断资源的 ETag 值是否发生变化。

看到这里相信你对示例中 `index.js` 资源请求返回 `304` 的原因，应该有了大致的理解。如果你对 `koa-etag` 中间件是如何生成 ETag 感兴趣的话，可以阅读 [HTTP 中的 ETag 是如何生成的？](https://mp.weixin.qq.com/s?__biz=MzI2MjcxNTQ0Nw==&mid=2247491652&idx=1&sn=6ece1636d4517a32bfb33845a5641f22&scene=21#wechat_redirect) 这篇文章。

### 四、缓存机制

强缓存优先于协商缓存进行，若强缓存（Expires 和 Cache-Control）生效则直接使用缓存，若不生效则进行协商缓存（Last-Modified/If-Modified-Since 和 Etag/If-None-Match），协商缓存由服务器决定是否使用缓存，若协商缓存失效，那么代表该请求的缓存失效，返回 200，重新返回资源和缓存标识，再存入浏览器缓存中；生效则返回 304，继续使用缓存。

具体的缓存机制如下图所示：

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/jQmwTIFl1V3smmqCwRiaxBVwRjNl6sLGAC0BeletDpcI8OJJcpItkOE1NXVDKmI3HuQ0pxAYbnAXrnjbs6MdeDw/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

为了让大家能够更好地理解缓存机制，我们再来简单分析一下前面的介绍 Koa 缓存示例：

```
// server.js
const Koa = require("koa");
const path = require("path");
const serve = require("koa-static");
const etag = require("koa-etag");
const conditional = require("koa-conditional-get");

const app = new Koa();

app.use(conditional()); // 使用条件请求中间件
app.use(etag()); // 使用etag中间件
app.use( // 使用静态资源中间件
  serve(path.join(__dirname, "/public"), {
    maxage: 10 * 1000, // 设置缓存存储的最大周期，单位为秒
  });
);

app.listen(3000, () => {
  console.log("app starting at port 3000");
});
```

以上示例使用了 `koa-conditional-get`、`koa-etag` 和 `koa-static` 这 3 个中间件。它们的具体定义分别如下：

#### 4.1 koa-conditional-get

```
// https://github.com/koajs/conditional-get/blob/master/index.js
module.exports = function conditional () {
  return async function (ctx, next) {
    await next()
    if (ctx.fresh) { // 资源未更新，则返回304 Not Modified 
      ctx.status = 304
      ctx.body = null
    }
  }
}
```

`koa-conditional-get` 中间件的实现很简单，如果资源是新鲜的，则直接返回 304 状态码并设置响应体为 null。

#### 4.2 koa-etag

```
// https://github.com/koajs/etag/blob/master/index.js
module.exports = function etag (options) {
  return async function etag (ctx, next) {
    await next()
    const entity = await getResponseEntity(ctx) // 获取响应实体对象
    setEtag(ctx, entity, options)
  }
}
```

在 `koa-etag` 中间件内部，当获取到响应实体对象之后，会调用 `setEtag` 函数来设置 ETag。`setEtag` 函数的定义如下：

```
// https://github.com/koajs/etag/blob/master/index.js
const calculate = require('etag')

function setEtag (ctx, entity, options) {
  if (!entity) return
  ctx.response.etag = calculate(entity, options)
}
```

很明显在 `koa-etag` 中间件内部是通过 `etag` 这个库，来为响应实体生成对应的 `etag` 的。

#### 4.3 koa-static

```
// https://github.com/koajs/static/blob/master/index.js
function serve (root, opts) {
  opts = Object.assign(Object.create(null), opts)
  // 省略部分代码
  return async function serve (ctx, next) {
    await next()
    if (ctx.method !== 'HEAD' && ctx.method !== 'GET') return
    // response is already handled
    if (ctx.body != null || ctx.status !== 404) return
    try {
      await send(ctx, ctx.path, opts)
    } catch (err) {
      if (err.status !== 404) {
        throw err
      }
    }
  }
}
```

对于 `koa-static` 中间件来说，当请求方法不是 GET 或 HEAD 请求（不应包含响应体）时，则直接返回。而静态资源的处理能力，实际是交由 `send` 这个库来实现的。

最后为了让小伙伴们能够更好地理解以上中间件的处理逻辑，阿宝哥带大家来简单回顾一下洋葱模型：

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

在上图中，洋葱内的每一层都表示一个独立的中间件，用于实现不同的功能，比如异常处理、缓存处理等。每次请求都会从左侧开始一层层地经过每层的中间件，当进入到最里层的中间件之后，就会从最里层的中间件开始逐层返回。因此对于每层的中间件来说，在一个 **请求和响应** 周期中，都有两个时机点来添加不同的处理逻辑。

