# axios源码解读

## 前言

为什么会想去读源码?

起因还是写小程序项目的时候，在回调里面总是写很多相同的判断，大大降低了我的代码体验感[🤷](https://emojipedia.org/shrug/)[🤷](https://emojipedia.org/shrug/)[🤷](https://emojipedia.org/shrug/)

`axios`自带拦截器，等等很多有用的方法，一直听闻axios实现很巧妙，虽然网络上已经有很对大佬的解读过axios，对我来说看文章不够尽兴，只有亲自探究`axios`的内部实现💪

## 从入口开始

下载了axios的源码

````
git clone https://github.com/axios/axios.git
````

就在我想这个源文件怎么运行的时候，我发现在文件夹里面似乎有示例代码`sandbox`里面有示例代码

运行后发现，这个是打包过的代码(陷入沉思.....)

因为浏览器不识别`require()` 此类的语法，所以直接应用源码会报错

于是

#### 搭建一个查看axios源码的环境

> 也就是支持es6的webpack配置(打包工具都可以，我对webpack比较熟悉)

这里解决的就是浏览器不认识**require**的问题，所以我们使用`webpack`，来搭建一个支持`es6`的开发环境，(还支持热加载，美滋滋~~~)

- 对搭建毫无压力的请直接跳过
- 想搭建环境的请看这里[搭建es6调试环境](https://github.com/vkcyan/text/blob/master/JavaScript/webpack%E6%90%AD%E5%BB%BAes6%E5%BC%80%E5%8F%91%E7%8E%AF%E5%A2%83.md)



## 目录结构

> 这里我引用自知乎[深入浅出 axios 源码](https://zhuanlan.zhihu.com/p/37962469)，目录结构写的清楚明了，前人栽树后人乘凉，我便直接引用了

```text
├── /lib/                      # 项目源码目
│ ├── /adapters/               # 定义发送请求的适配器
│ │ ├── http.js                # node环境http对象
│ │ └── xhr.js                 # 浏览器环境XML对象
│ ├── /cancel/                 # 定义取消功能
│ ├── /helpers/                # 一些辅助方法
│ ├── /core/                   # 一些核心功能
│ │ ├── Axios.js               # axios实例构造函数
│ │ ├── createError.js         # 抛出错误
│ │ ├── dispatchRequest.js     # 用来调用http请求适配器方法发送请求
│ │ ├── InterceptorManager.js  # 拦截器管理器
│ │ ├── mergeConfig.js         # 合并参数
│ │ ├── settle.js              # 根据http响应状态，改变Promise的状态
│ │ └── transformData.js       # 改变数据格式
│ ├── axios.js                 # 入口，创建构造函数
│ ├── defaults.js              # 默认配置
│ └── utils.js                 # 公用工具
```



## axios的运行流程

我阅读完成源码后，梳理了一下运行思路，axios大致分为 **四个阶段**

1. 初始化参数
2. 初始化拦截器，添加原型链方法
3. 合并原型链对象到axios，便于直接调用
4. 开始进行请求(有拦截器执行拦截器)



下面将结合代码以及图片进行axios内部实现过程的梳理



当我们开始使用axios的时候，例如

```js
let axios = require('./lib/axios')

axios('https://www.easy-mock.com/mock/.....')
  .then(res => {
    // .....
  })
  .catch(res => {
    // .....
  })

```

使用了`./lib/axios`导出的axios对象，毫无疑问这是axios的入口文件

## 初始化参数

**lib/axios**

> 为了便于查看，我删除了一些axios其他方法的代码

````js
'use strict'

var utils = require('./utils')
var bind = require('./helpers/bind')
var Axios = require('./core/Axios')
var mergeConfig = require('./core/mergeConfig')
var defaults = require('./defaults')

/**
 * 创建Axios实例
 *
 * @param {Object} defaultConfig实例的默认配置
 * @return {Axios} Axios的新实例
 */
function createInstance(defaultConfig) { // (2)
  // defaultConfig 为初始化配置
  var context = new Axios(defaultConfig);

  // bind(Axios.prototype.request， context)的目的是让axios默认及request方法
  var instance = bind(Axios.prototype.request， context)
  
  // 将Axios.prototype的原型链上的对象复制到instance上面
  utils.extend(instance， Axios.prototype， context);
    
  // 将拦截器，默认配置复制到instance(这里主要是拦截器操作)
  utils.extend(instance， context)
    
  return instance
}

// 创建要导出的默认实例
// 当axios执行的时候 首先axios进行默认的初始化配置
var axios = createInstance(defaults) // (1)

module.exports = axios

// 允许在TypeScript中使用默认导入语法
module.exports.default = axios
````



这里可以看到，当我们执行`axios`的时候，`axios =createInstance(defaults)`，也就是说 `axios`是`createInstance()`

于是执行`createInstance(defaults)` `default`是其他文件引入，用于参数初始化

> 因为default的代码很长，这里就不贴了，这里说明初始化都做了什么

在`lib/default.js`

1. 判断当前是浏览器环境还是`node`环境，分别使用`xhr`与 `http`
2. 添加默认配置，例如`header` `itemout`
3. 帮助我们默认添加`xsrfCookieName`，用于防止`asrf`攻击
4. 等等很多初始化参数

##### 完成初始化

![img](http://www.vkcyan.top/FiAhMcl7leBqc_BseYHRpkT3ts3J.png)



## 初始化拦截器，添加原型链方法

当参数初始化完成后，被作为参数传递到`createInstance()`

```
// defaultConfig 为初始化配置
var context = new Axios(defaultConfig)
```

执行了`./lib/core/Axios.js`

### 初始化拦截器

```js
// ...... 导入库

/**
 * 创建Axios的新实例
 *
 * @param {Object} instanceConfig实例的默认配置
 */
function Axios(instanceConfig) {
  // eslint-disable-next-line no-console
  this.defaults = instanceConfig // 将初始化的参数挂载到this上 便于原型的访问
  this.interceptors = {
    request: new InterceptorManager()，
    response: new InterceptorManager()
  }
}

// ...... 等等原型链代码
```

这里，可以看到`this.interceptors `，是不是很眼熟?这是我们使用的拦截器对象

所以需要看看，初始化的时候，拦截器内部是什么

`./lib/core/InterceptorManager`

```js
'use strict';

var utils = require('./../utils');

function InterceptorManager() {
  this.handlers = [];
}

InterceptorManager.prototype.use = function use(fulfilled， rejected) {
  this.handlers.push({
    fulfilled: fulfilled，
    rejected: rejected
  });
  // 这里返回没问题 但是为什么这么写?
  return this.handlers.length - 1;
};


InterceptorManager.prototype.eject = function eject(id) {
  if (this.handlers[id]) {
    this.handlers[id] = null;
  }
};

// 迭代所有已注册的拦截器
InterceptorManager.prototype.forEach = function forEach(fn) {
  // fn > function
  utils.forEach(this.handlers， function forEachHandler(h) {
    if (h !== null) {
      // 将拦截器里面的函数交给其回调
      fn(h);
    }
  });
};

module.exports = InterceptorManager;

```

似乎简单的出乎意料，仅仅是初始化了一个数组，提供了`use(push数组)` `eject(清空数组)` `forEach`三个方法

new Axios完成后(原型方法还没有挂载前)

![](http://www.vkcyan.top/FoaUkQVyyYVhsLEOwdwUOB_vwIQt.png)

拦截器初始化完成了

### 初始化原型方法

> 这里是axios实现的核心

`./lib/core/Axios.js`

```js
// ...... new Axios().... 



/**
 * 发送请求方法
 *
 * @param {Object} config The config specific for this request (merged with this.defaults)
 */
// 当我们请求`axios.request(//...) 参数就会Axios.prototype.request接收        
Axios.prototype.request = function request(config) {
  /*eslint no-param-reassign:0*/
  // 作者为我们提供axios('url'，{})以及axios(config)两种写法
  if (typeof config === 'string') {
    config = arguments[1] || {} 
    config.url = arguments[0]
  } else {
    config = config || {}
  }
  // 合并 defaultconfig与自定义的config
  config = mergeConfig(this.defaults， config)
  config.method = config.method ? config.method.toLowerCase() : 'get' // 默认没有都是get请求

  // 创造一个请求序列数组 第一位是发送请求的方法，第二位是空
  var chain = [dispatchRequest， undefined]
  var promise = Promise.resolve(config) // 创建一个promise对象，并把参数传递进去
  //this.interceptors.request = InterceptorManager() 如果定义拦截器 那么 这里的InterceptorManager内部的handlers就会存在你写的拦截器代码
  // 执行InterceptorManager原型链上的forEach事件
  this.interceptors.request.forEach(function unshiftRequestInterceptors(
    interceptor
  ) {
    // interceptor 为 执行InterceptorManager原型链上的forEach事件返回的 拦截器函数
    //把请求拦截器数组依从加入头部
    chain.unshift(interceptor.fulfilled， interceptor.rejected)
  })

  this.interceptors.response.forEach(function pushResponseInterceptors(
    interceptor
  ) {
    // 同理
    // 将接收拦截器数组一次加入尾部
    chain.push(interceptor.fulfilled， interceptor.rejected)
  })

  while (chain.length) {
    // shift() 方法用于把数组的第一个元素从其中删除，并返回第一个元素的值。
    // 形成一个promise调用链条
    promise = promise.then(chain.shift()， chain.shift())
  }
  return promise
}

Axios.prototype.getUri = function getUri(config) {
  config = mergeConfig(this.defaults， config)
  return buildURL(config.url， config.params， config.paramsSerializer).replace(
    /^\?/，
    ''
  )
}

// 为支持的请求方法提供别名
utils.forEach(
  ['delete'， 'get'， 'head'， 'options']，
  function forEachMethodNoData(method) {
    /*eslint func-names:0*/
    // 这里提供了语法通，在axios的原型链条上面 增加'delete'， 'get'， 'head'， 'options'直接调用的方法 实际上还是request
    Axios.prototype[method] = function(url， config) {
      return this.request(
        utils.merge(config || {}， {
          method: method，
          url: url
        })
      )
    }
  }
)

// 与上方同理
utils.forEach(['post'， 'put'， 'patch']， function forEachMethodWithData(method) {
  /*eslint func-names:0*/
  Axios.prototype[method] = function(url， data， config) {
    return this.request(
      utils.merge(config || {}， {
        method: method，
        url: url，
        data: data
      })
    )
  }
})

module.exports = Axios
```

这里的代码，比较晦涩难懂，但是实现的非常巧妙，我用以下文图来表达



当我们执行以下代码

```js
let axios = require('./lib/axios')
axios.interceptors.request.use(
  function(config) {
    console.log(config);
    return config
  }，
  function(error) {
    return Promise.reject(error)
  }
)
axios.interceptors.response.use(
  function(res) {
    console.log(res);
    return res
  }，
  function(error) {
    console.log(error);
  }
)

console.log('请求开始前的准备个工作');
axios.get('......')
  .then(res => {
    console.log(res)

  })
  .catch(res => {
    console.log(res)
  }) 
```

![](http://www.vkcyan.top/FkYrFRVTFzKtYrh3HxyBHPQB6zaY.png)



这里由请求序列数组实现的`promise`执行链条，帮助我们实现了`拦截器`等功能

借用知乎[熵与单子的代码本](https://zhuanlan.zhihu.com/c_90568250)大佬的图示(我觉得简单明了)

 ![](https://pic3.zhimg.com/80/v2-3445b621a1e8214b0b3816ddcabfab86_hd.png)



再次引用[熵与单子的代码本](https://zhuanlan.zhihu.com/c_90568250)的[【源码拾遗】axios —— 极简封装的艺术](https://zhuanlan.zhihu.com/p/28396592)的一句话

```
通过巧妙的利用unshift、push、shift等数组队列、栈方法，实现了请求拦截、执行请求、响应拦截的流程设定，注意无论是请求拦截还是响应拦截，越先添加的拦截器总是越“贴近”执行请求本身。
```





## 对原型链上的方法的处理

不知 道目前为止 看起来似乎一起变的比较明了，但是还存在一些问题

> request get post 等等方法都在Axios.prototype上面，也就是说目前为止还无法直接调用原型链上的方法

于是我们看**./lib/axios.js**的 `createInstance(defaultConfig) `后面还没有看的代码

```JavaScript
function createInstance(defaultConfig) {
  console.log(defaultConfig);
  
  // defaultConfig 为初始化配置
  var context = new Axios(defaultConfig)
  // 将Axios.prototype.request的this传递给context
  // bind(Axios.prototype.request， context)的目的是让axios默认及request方法
  var instance = bind(Axios.prototype.request， context)
  // var instance = context
  // 将Axios.prototype的原型链上的对象复制到instance上面
  utils.extend(instance， Axios.prototype)
  // utils.extend(instance， Axios.prototype， context)
  // 将拦截器，默认配置复制到instance(这里主要是拦截器操作)
  utils.extend(instance， context)
  return instance
}
```

着重看这几行

```JavaScript
  var instance = bind(Axios.prototype.request， context)
  utils.extend(instance， Axios.prototype， context)
  utils.extend(instance， context)

  return instance
```

这里用到了`bind`，所以我们还要看看bind

```js
module.exports = function bind(fn， thisArg) {
  return function wrap() {
    // 获取到axios(//...)里面的参数
    var args = new Array(arguments.length)
    for (var i = 0; i < args.length; i++) {
      args[i] = arguments[i]
    }
    // 将fn的this传递给thisArg 例如将axios('www.baidu.com') 就会执行这个warp函数最后将参数交给fn也就是Axios.prototype.request
    return fn.apply(thisArg， args)
  }
}
```

明显看出 经过处理的`instance`是一个对象

当我们使用

```
axios('// .... 请求连接')
```

就是执行`wrap()`，将参数，全部`apply`到fn上，也就是`Axios.prototype.request`上，这样的this指向的调换，即默认

```
axios('//....') === Axios.prototype.request('//....')
```

后面的extend，帮助我们将其他Axios的原型属性`get` `post` 等等，移植到返回出去的`instance`上面

```
axios.get('//....') === Axios.prototype.get('//....')
```

否则我们无法通过`axios.get()` `axios.post`进行调用

```js
/**
 * 为对象a拓展对象b的属性
 *
 * @param {Object} a 需要拓展的属性
 * @param {Object} b 要从中复制属性的对象
 * @param {Object} thisArg 要绑定函数的对象
 * @return {Object} The 返回值
 */
function extend(a， b， thisArg) {
  console.log(b);
  forEach(b， function assignValue(val， key) {
    if (thisArg && typeof val === 'function') {
      a[key] = bind(val， thisArg); // 一般情况下 和 else 效果 我不是很明白这里bind的必要性
    } else {
      a[key] = val; // 将b的属性移植到 a上
    }
  });
  return a;
}
```

最后返回出去的`instance`，就是我们日常使用的axios了 😃



## 请求流程

最后根据上面的总结，梳理一下axios的请求过程

![](http://www.vkcyan.top/Fm6-VsddRCNRQUkNaA90OwVnZR66.png)

## 总结

​	最近不是很忙，所以抽出时间，查看了axois的源码，本文仅仅很粗略的理了axios的执行过程，以及内部的一些实现思路，实际上axios源码的内容非常丰富，也还有很多模块没有说，例如 **adapter** **dispatchRequest**，**取消请求**，等等，还有很多 **工具函数** 值得我们学习



​	可能有些开发者也有想去读源码的心情，但是很多库文件之间**依赖关系复杂**，仅仅去读 是很难读懂的，所以，我们需要把源码运行起来，在关键地方`debugger`以及添加`log`更加容易理清思路.那么去clone我的环境[]()，进行axios源码的阅读



​	相关文章已经有很多大佬解读，本篇文章为个人总结，，如有错误，或者歧义 欢迎提出 欢迎交流🙈🙈🙈