# Axios 源码

> 几款热门 HTTP 请求库在 GitHub 上的受欢迎程度

| 热门 JS HTTP 请求库 |            特性简介             | Star  | Fork |
| :-----------------: | :-----------------------------: | :---: | :--: |
|        Axios        | 基于 Promise，支持浏览器和 node | 85.4k | 8.3k |
|       Request       |  不基于 Promise，简化版的 HTTP  | 25.2k | 3.1k |
|        Fetch        | 基于 Promise，不支持 node 调用  | 24.8k |  3k  |
|     Superagent      |                                 | 15.7k | 1.3k |

虽然大家都是对 XMLHttpRequest 的封装，但是纵观 Axios 的热度，一骑绝尘啊！由此可见，Axios 真的是一个很优秀的开源项目。然而惭愧的是日常开发中总是拿来就用，一直没有静下心来好好拜读一番 Axios 的源码，会不会有很多人跟我一样呢？这里先列举一下 axios 项目的核心目录结构：

```
lib

└─ adapters

   ├─ http.js // node 环境下利用 http 模块发起请求

   ├─ xhr.js // 浏览器环境下利用 xhr 发起请求

└─ cancel

   ├─ Cancel.js

   ├─ CancelToken.js

   ├─ isCancel.js

└─ core

    ├─ Axios.js // 生成 Axios 实例

    ├─ InterceptorManager.js // 拦截器

    ├─ dispatchRequest.js  // 调用适配器发起请求

    ...

└─ helpers

    ├─ mergeConfig.js // 合并配置

    ├─ ...

├─ axios.js  // 入口文件

├─ defaults.js  // axios 默认配置项

├─ utils.js
```

## 简介

Axios 是一个基于 Promise 网络请求库，作用于 node.js 和浏览器中。在服务端它使用原生 node.js`http`模块, 而在客户端 (浏览端) 则使用 XMLHttpRequests。特性：

- 从浏览器创建XMLHttpRequests

- 从 node.js 创建http请求
- 支持PromiseAPI
- 拦截请求和响应
- 转换请求和响应数据
- 取消请求
- 自动转换 JSON 数据
- 客户端支持防御XSRF

## Axios 内部运作流程

![图片](https://mmbiz.qpic.cn/mmbiz_png/lP9iauFI73zicqIKtS2GZJAscn2mSicDf8VHXnF5rXUIaf7IBibRPbVFZp9YRnS9YQ4SVSpCojicnrvPibpkXuqlorug/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)接下来我们结合 axios 的运作流程一起来剖析以下几个模块：

- Axios 构造函数
- 请求 / 响应拦截器
- dispatchRequest 派发请求
- 转换请求 / 响应数据
- 适配器处理 HTTP 请求

## Axios 如何支持不同的使用方式?

### 使用 axios 发起请求

我们先来回忆一下平时是如何使用 axios 的：

```
// 方式 1  axios(config)

axios({

    method: 'get',

    url: 'xxx',

    data: {}

});



// 方式 2  axios(url[, config]),默认 get 请求

axios('http://xxx');



// 方式 3 使用别名进行请求

axios.request(config)

axios.get(url[, config])

axios.post(url[, data[, config]])

axios.put(url[, data[, config]])

...



// 方式 4 创建 axios 实例，自定义配置

const instance = axios.create({

  baseURL: 'https://some-domain.com/api/',

  timeout: 1000,

  headers: {'X-Custom-Header': 'foobar'}

});



axios#request(config)

axios#get(url[, config])

axios#post(url[, data[, config]])

axios#put(url[, data[, config]])

...
```

### 源码分析

首先来看 axios 的入口文件， lib 目录下的`axios.js`:

```
// /lib/axios.js

function createInstance(defaultConfig) {

  // 创建 axios 实例

  var context = new Axios(defaultConfig);

  // 把 instance 指向 Axios.prototype.request 方法

  var instance = bind(Axios.prototype.request, context);

  // 把 Axios.prototype 上的方法扩展到 instance 上，指定上下文是 context

  utils.extend(instance, Axios.prototype, context);

  // 把 context 上的方法扩展到 instance 上

  utils.extend(instance, context);

  // 导出 instance 对象

  return instance;

}

var axios = createInstance(defaults);

// 添加 create 方法，返回 createInstance 函数，参数为自定义配置 + 默认配置

axios.create = function create(instanceConfig) {

  return createInstance(mergeConfig(axios.defaults, instanceConfig));

};



...



module.exports = axios;

// Allow use of default import syntax in TypeScript

module.exports.default = axios;
```

可见，当我们调用`axios()`时，实际上是执行了`createInstance`返回的一个指向`Axios.prototype.request`的函数；通过添加`create`方法支持用户自定义配置创建，并且最终也是执行了`Axios.prototype.request`方法；接下来我们看看`Axios.prototype.request`的源码是怎么写的：

```
// /lib/core/Axios.js

// 创建一个 Axios 实例

function Axios(instanceConfig) {

  ...

}

Axios.prototype.request = function request(config) {

  // 判断 config 类型并赋值

  // 方式二：axios('https://xxxx') ，判断参数字符串，则赋值给 config.url

  if (typeof config === 'string') {

    config = arguments[1] || {};

    config.url = arguments[0];

  } else {

  // 方式一：axios({}) ,参数为对象，则直接赋值给 config

    config = config || {};

  }

  ...

}

...

// 方式三 & 方式四

// 遍历为请求设置别名

utils.forEach(['delete', 'get', 'head', 'options'], function forEachMethodNoData(method) {

  /*eslint func-names:0*/

  Axios.prototype[method] = function(url, config) {

    return this.request(mergeConfig(config || {}, {

      method: method,

      url: url,

      data: (config || {}).data

    }));

  };

});

// 遍历为请求设置别名

utils.forEach(['post', 'put', 'patch'], function forEachMethodWithData(method) {

  /*eslint func-names:0*/

  Axios.prototype[method] = function(url, data, config) {

    return this.request(mergeConfig(config || {}, {

      method: method,

      url: url,

      data: data

    }));

  };

});
```

到此，axios 支持了 4 中不同的使用方式，无论哪种使用方式，最后都是执行 Axios 实例上的核心方法：`request`。

## 请求 / 响应拦截器是如何生效的？

### 设置拦截器

对于大多数 spa 的项目来说，通常会使用 token 进行用户的身份认证，这就要求每个请求都携带认证信息；接收到服务器信息之后，如果发现用户未登录，需要统一跳转登录页；遇到这种场景，就需要用到 axios 提供的拦截器，以下是拦截器的设置：

```
 // 添加请求拦截器

axios.interceptors.request.use(function (config) {

  config.headers.token = 'xxx';

  return config;

});



 // 添加响应拦截器

axios.interceptors.response.use(function (response) {

    if(response.code === 401) {

        login()

    }

    return response;

});
```

### 源码分析

通过拦截器的使用，可以知道实例 Axios 上添加了`interceptors`方法，接下来我们看看源码的实现：

```
// /lib/core/Axios.js

// 每个 Axios 实例上都有 interceptors 属性，该属性上有 request、response 属性，

// 分别都是一个 InterceptorManager 实例，而 InterceptorManager 构造函数就是

// 用来管理拦截器

function Axios(instanceConfig) {

  this.defaults = instanceConfig;

  this.interceptors = {

    request: new InterceptorManager(),

    response: new InterceptorManager()

  };

}



// /lib/core/InterceptorManager.js

function InterceptorManager() {

  this.handlers = []; // 拦截器

}

// 往拦截器里 push 拦截方法

InterceptorManager.prototype.use = function use(fulfilled, rejected, options) {

  this.handlers.push({

    fulfilled: fulfilled,

    rejected: rejected,

    ...

  });

  // 返回当前索引，用于注销指定拦截器

  return this.handlers.length - 1;

};
```

Axios 与`InterceptorManager`的关系如图示：![图片](https://mmbiz.qpic.cn/mmbiz_png/lP9iauFI73zicqIKtS2GZJAscn2mSicDf8VAhCFvYRmaEMicheYTKTibUaxW2icibBLicS6WvWiamQcM3W2lZv1XIS1dw7g/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)现在我们已经有了拦截器，那么 axios 是如何保证发起请求的顺序执行呢？

- 请求拦截器 => http 请求 => 响应拦截器

上源码：

```
// /lib/core/Axios.js

// request 方法中

// 省略部分代码

// 生成请求拦截队列

var requestInterceptorChain = [];

this.interceptors.request.forEach(function unshiftRequestInterceptors(interceptor) {

    requestInterceptorChain.unshift(interceptor.fulfilled, interceptor.rejected);

});

// 生成响应拦截队列

var responseInterceptorChain = [];

this.interceptors.response.forEach(function pushResponseInterceptors(interceptor) {

    responseInterceptorChain.push(interceptor.fulfilled, interceptor.rejected);

});



// 编排整个请求的任务队列

var chain = [dispatchRequest, undefined];

Array.prototype.unshift.apply(chain, requestInterceptorChain);

chain.concat(responseInterceptorChain);



promise = Promise.resolve(config);

// 循环 chain ，不断从 chain 中取出设置的任务，通过 Promise 调用链执行

while (chain.length) {

  promise = promise.then(chain.shift(), chain.shift());

}



return promise;
```

用图示表示一下拦截器过程更清晰：![图片](https://mmbiz.qpic.cn/mmbiz_png/lP9iauFI73zicqIKtS2GZJAscn2mSicDf8VjjaoC6auGdLcic9UyIpFU1srpJtMR2LzKhrTPic1aic9hrbUpjGAUSkAg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)生成任务队列后，再通过`promise.then(chain.shift(), chain.shift())`调用 Promise 链去处理设置的任务。这里需要注意一点，请求拦截队列在生成时，是通过`Array.unshift(fulfilled, rejected)`设置的，也就是说在执行请求拦截时，先设置的拦截方法后执行，后设置的拦截方法先执行。

## 派发请求 dispatchRequest

### 源码分析

处理完请求拦截之后，总算开始步入整个请求链路的正轨，也就是上图中任务队列的中间步骤：`dispatchRequest`派发请求。

```
// /lib/core/dispatchRequest.js

module.exports = function dispatchRequest(config) {

  // 转换请求数据

  config.data = transformData.call(

    config,

    config.data,

    config.headers,

    config.transformRequest

  );

  ...

  // 适配器 可以自定义适配器，没有自定义，执行axios默认适配器

  var adapter = config.adapter || defaults.adapter;

  // 通过适配器处理 config 配置，返回服务端响应数据 response

  return adapter(config).then(function onAdapterResolution(response) {

    ...

    // 转换响应数据

    response.data = transformData.call(

      config,

      response.data,

      response.headers,

      config.transformResponse

    );

    ...

    return response;

  }, function onAdapterRejection(reason) {

    ...

    return Promise.reject(reason);

  }）

}
```

`dispatchRequest`中主要做了两件事，先通过`transformData`对请求数据进行处理，然后定义适配器`adapter`并执行，通过 .then 方法 对`adapter`（适配器） resolve 出的响应数据进行处理（`transformData`）并返回 response，失败返回一个状态为`rejected``的 Promise 对象。到此也就明白，当用户调用 axios()时，为什么可以链式调用 Promise 的 .then() 和 .catch() 来处理业务逻辑了。接下来我们从`transformData`入手，看看 axios 是如何转换请求和响应数据的。

## 转换请求 / 响应数据

### 源码分析

```
// /lib/core/dispatchRequest.js

config.data = transformData.call(

    config,

    config.data,

    config.headers,

    config.transformRequest

);



// /lib/core/transformData.js

module.exports = function transformData(data, headers, fns) {

    utils.forEach(fns, function transform(fn) {

    data = fn(data, headers);

    });



    return data;

};
```

通过上述代码可以发现`transformData`方法主要是遍历`config.transformRequest`数组中的方法，`config.data`和`config.headers`作为参数。来看一下`transformRequest`和`tranformResponse`的定义:

```
// /lib/default.js

var default = {

  ...

  // 转换请求数据

  transformRequest: [function transformRequest(data, headers) {

    // 判断 data 类型

    if (utils.isFormData(data) ||

      utils.isArrayBuffer(data) ||

      utils.isBuffer(data) ||

      utils.isStream(data) ||

      utils.isFile(data) ||

      utils.isBlob(data)

    ) {

      return data;

    }

    if (utils.isArrayBufferView(data)) {

      return data.buffer;

    }

    if (utils.isURLSearchParams(data)) {

      setContentTypeIfUnset(headers, 'application/x-www-form-urlencoded;charset=utf-8');

      return data.toString();

    }

    // 如果 data 是对象，或 Content-Type 设置为 application/json

    if (utils.isObject(data) || (headers && headers['Content-Type'] === 'application/json')) {

      // 设置 Content-Type 为 application/json

      setContentTypeIfUnset(headers, 'application/json');

      // 将 data 转换为 json 字符串返回

      return JSON.stringify(data);

    }

    return data;

  }],

  // 转换响应数据

  transformResponse: [function transformResponse(data) {

    ...

    if (strictJSONParsing || (forcedJSONParsing && utils.isString(data) && data.length)) {

      try {

        // 将 data 转换为 json 对象并返回

        return JSON.parse(data);

      } catch (e) {

        ...

      }

    }

    return data;

  }],

  ...

}
```

到此，请求数据和响应数据的转换过程已经结束了，顺便提一下，官方文档介绍的特性之一：**自动转换 JSON 数据**，应该就是转换过程中的`JSON.stringify(data)`与`JSON.parse(data)`了;

### 重写 / 新增转换方法

发现`transformRequest`方法是`default`对象上的一个属性，那么我们是不是可以通过自定义配置来改写转换的过程呢？

```
import axios from 'axios';

// 重写转换请求数据的过程

axios.default.transformRequest = [(data, headers) => {

    ...

    return data

}];

// 增加对请求数据的处理

axios.default.transformRequest.push(

(data, headers) => {

    ...

    return data

});
```

## 适配器（adapter）处理请求

`dispatchRequest`方法做的第二件事：定义`adapter`，并执行。接下来，我们来揭开`adapter`的面纱，看看它具体是怎么处理 HTTP 请求的~

### 源码分析

下面的代码可以看出，适配器是可以自定义的，如果没有自定义，则执行 axios 提供的默认适配器。

```
// /lib/core/dispatchRequest.js (51行)

var adapter = config.adapter || defaults.adapter;
```

我们先来分析默认适配器，在`default.js`中：

```
function getDefaultAdapter() {

    var adapter;

    // 判断当前环境

    if (typeof XMLHttpRequest !== 'undefined') {

    // 浏览器环境，使用 xhr 请求

    adapter = require('./adapters/xhr');

    } else if (typeof process !== 'undefined' && Object.prototype.toString.call(process) === '[object process]') {

    // node 环境，使用 http 模块

    adapter = require('./adapters/http');

    }

    return adapter;

}

var defaults = {

    ...

    // 定义 adapter 属性

    adapter: getDefaultAdapter(),

    ...

}
```

可以看到，axios 之所以支持浏览器环境和 node 环境，就是`getDefaultAdapter`方法进行了环境判断，分别使用**xhr 处理浏览器请求**和**http 模块处理 node 请求**。官方称之为`isomorphic`（同构）能力。这里定义了`defaults`对象，该对象定义了 axios 的一系列默认配置，还记得它是在哪被注入到 axios 中的吗？当然是在入口文件`axios.js`里了。

```
// /lib/axios.js

...

var defaults = require('./defaults');

...

function createInstance(defaultConfig) {

  ...

  // 创建 axios 实例

  var context = new Axios(defaultConfig);

  ...

}

var axios = createInstance(defaults);

...
```

哎呦，串起来了有没有~好的，重新说回到 xhr 请求，本文只分析浏览器环境中 axios 的运行机制，因此接下来，让我们打开`./adapters/xhr`文件来看一下：

```
module.exports = function xhrAdapter(config) {

  return new Promise(function dispatchXhrRequest(resolve, reject) {

    ...

    var request = new XMLHttpRequest();

    // 设置完整请求路径

    var fullPath = buildFullPath(config.baseURL, config.url);

    request.open(config.method.toUpperCase(), buildURL(fullPath, config.params, config.paramsSerializer), true) ;

    // 请求超时

    request.timeout = config.timeout;

    request.ontimeout = function handleTimeout() {...}

    // 请求中断

    request.onabort = function handleAbort() {...}

    ...

    request.send(requestData);

  }

}
```

将 config 中的请求配置进行赋值处理，正式发起`XMLHttpRequest`请求。

### 自定义 adapter

通过上面对 adapter 的分析，可以发现如果自定义 adapter 的话，是可以接管 axios 的请求和响应数据的，因此可以自定义 adapter 实现 mock；

```
const mockUrl = {

    '/mock': {data: xxx}

};

const instance = Axios.create({

    adapter: (config) => {

        if (!mockUrl[config.url]) {

            // 调用默认的适配器处理需要删除自定义适配器，否则会死循环

            delete config.adapter

            return Axios(config)

        }

        return new Promise((resolve, reject) => {

            resolve({

                data: mockUrl[config.url],

                status: 200,

            })

        })

    }

})
```

