# koa源码阅读

<a name="04c2d6bf"></a>
## KOA用法
   koa 的用法十分简单，以官网的 'hello world' 为例<br /> 
```javascript
const Koa = require('koa');
const app = new Koa();

app.use(async ctx => {
  ctx.body = 'Hello World';
});

app.listen(3000);
```

对比原生

```javascript
let http = require('http')

let server = http.createServer((req, res) => {
  res.end('hello world')
})

server.listen(4000)
```

对比发现，koa 多了两个实例上的 use 和 listen 方法，和 use 回调中的 ctx 参数，这几个不同，几乎就是koa的全部。

<a name="5b804b05"></a>
## 源码

koa 的源码很简单，主要代码只有4个文件，分别是lib下面的 application.js , context.js . request.js ,response.js 。通过查看 package.json 可以发现， application.js 为入口文件。<br />![image.png](https://cdn.nlark.com/yuque/0/2019/png/206908/1555490911053-1f863e5e-9ab9-4772-b372-af114cab3f00.png#align=left&display=inline&height=77&name=image.png&originHeight=77&originWidth=416&size=6911&status=done&width=416)

那么，我们首先来看一下 application.js 这个文件。

```javascript
const Emitter = require('events');
const http = require('http');
const compose = require('koa-compose');
const response = require('./response');
const context = require('./context');
const request = require('./request');

module.exports = class Application extends Emitter {
  constructor() {
    super();
    this.middleware = [];
    this.context = Object.create(context);
    this.request = Object.create(request);
    this.response = Object.create(response);
     
    listen(...args) {
      const server = http.createServer(this.callback());
      return server.listen(...args);
    }
    
    use(fn) {
      this.middleware.push(fn);
      return this;
    }
    
    
  callback() {
    const fn = compose(this.middleware);
    const handleRequest = (req, res) => {
      const ctx = this.createContext(req, res);
      return this.handleRequest(ctx, fn);
    };
    return handleRequest;
   }

  }
```
 <br />结合上面 hello world 的例子，看出来 koa 是一个类，为了监听实例的 error 事件，所以要引入继承 events 模块。

实例上主要有两个方法，一个是 use ，另一个就是 listen 。从例子可以看出， 我们在使用 use 方法时，传进来的方法会被push进 middleware 这个数组存起来，在 listen 的时候，创建了 createContext 函数用来创建上下文，并创建 handleRequest 函数处理请求，并将 handleRequest 放进 createServe 的回调中，用户就得到了ctx 。那么，重点就在 compose 以及 createContext 这两个方法中。我们首先来看 createContext 这个函数 

<a name="createContext"></a>
### createContext

```javascript
  createContext(req, res) {
    // 使用Object.create方法是为了继承this.context但在增加属性时不影响原对象
    const context = Object.create(this.context);
    const request = context.request = Object.create(this.request);
    const response = context.response = Object.create(this.response);
    context.app = request.app = response.app = this;
    context.req = request.req = response.req = req;
    context.res = request.res = response.res = res;
    request.ctx = response.ctx = context;
    request.response = response;
    response.request = request;
    context.originalUrl = request.originalUrl = req.url;
    context.state = {};
    return context;
  }

```

可以看到，这个函数的功能很简单,,就是继承了用 context ，request ,response 继承了 context.js  , request.js ,response.js 三个文件导出的对象，然后经常一系列的操作使用户可以用各种姿势取到想要的值。例如url,为了使用ctx.req.url , ctx.request.req.url , ctx.reqsponse.req.url , ctx.url 都能取到值并能拓展出了原生以外的属性，在request.js 中可以看到    <br />

```javascript
const URL = require('url').URL;

module.exports = {
	  get url() {
    return this.req.url;
  },	
}
```

很简单，在 request.js 这个文件使用 get 返回一个经过处理的数据就可以将数据绑定到 request 上了，由于前面一系列的操作。 this 就是 ctx ,所以this.req.url 就是 ctx.req.url，那么 this.req 就是 ctx.req 也就是原生的 req 。这样还不能用 ctx.url 取值，于是在 context 中引用了 delegate 模块进做了一个代理。delegate 模块代码如下

```javascript
function Delegator(proto, target) {
  if (!(this instanceof Delegator)) return new Delegator(proto, target);
  this.proto = proto;
  this.target = target;
  this.methods = [];
  this.getters = [];
  this.setters = [];
  this.fluents = [];
}

Delegator.prototype.getter = function(name){
  var proto = this.proto;
  var target = this.target;
  this.getters.push(name);
  proto.__defineGetter__(name, function(){
    return this[target][name];
  });
  return this;
};

```
 <br />context 文件就是 proto ,每一个对象下面都有一个__defineGetter__方法，所以当你使用 ctx.url 时，就触发了__defineGetter__方法，这里的 this 就是 ctx，target 就是 reqsuest ,所以返回的就是 ctx.requset.url。

同样，因为原生是使用res.end(xx)，但是 koa 的 api 使用 ctx.body 。所以还要实现设置和获取 ctx.body。<br />我们来看看 response.js

```javascript
module.exports = {
	 get body() {
    return this._body;
  }
}
```

<br />同样，这里获取的是 ctx.response.body ,并不是 ctx.body ,经过 delegate 模块代理一下就可以获得 ctx.body了。<br />另外，一旦给 body 设置值，状态码就会变成 200，用户输入文件，json等文件时，我们都要处理。所以在application.js 中 有一个专门处理的函数 response

```javascript
function respond(ctx) {
  // allow bypassing koa
  if (false === ctx.respond) return;

  if (!ctx.writable) return;

  const res = ctx.res;
  let body = ctx.body;
  const code = ctx.status;

  // ignore body
  if (statuses.empty[code]) {
    // strip headers
    ctx.body = null;
    return res.end();
  }

  if ('HEAD' == ctx.method) {
    if (!res.headersSent && isJSON(body)) {
      ctx.length = Buffer.byteLength(JSON.stringify(body));
    }
    return res.end();
  }

  // status body
  if (null == body) {
    if (ctx.req.httpVersionMajor >= 2) {
      body = String(code);
    } else {
      body = ctx.message || String(code);
    }
    if (!res.headersSent) {
      ctx.type = 'text';
      ctx.length = Buffer.byteLength(body);
    }
    return res.end(body);
  }

  // responses
  if (Buffer.isBuffer(body)) return res.end(body);
  if ('string' == typeof body) return res.end(body);
  if (body instanceof Stream) return body.pipe(res);

  // body: json
  body = JSON.stringify(body);
  if (!res.headersSent) {
    ctx.length = Buffer.byteLength(body);
  }
  res.end(body);
}

```

<a name="compose"></a>
### compose

这样createContext函数我们就整体了解。接下来我们来看一下compose.

在 hello world 的例子中，我们只使用了一次 use，其实 use 是可以多次使用的，为了实现多次使用，并在一个从一个中间件跳到另一个中间, koa 提出了'洋葱模型' .

首先我们来写一段代码

```javascript
const Koa = require("koa");

const app = new Koa();

app.use(async (ctx, next) => {
  console.log(1);
  await next();
  console.log(2);
});

app.use(async (ctx, next) => {
  console.log(3);
  await next();
  console.log(4);
});

app.use(async (ctx, next) => {
  console.log(5);
  await next();
  console.log(6);
});

app.listen(3001);

// 1
// 3
// 5
// 6
// 4
// 2
```

next方法会调用下一个use，next下面的代码会在下一个use执行完再执行,一直到最后，所以输出的顺序就是1,3,5,6,4,2。接下来我们来看实现方法

```javascript
function compose (middleware) {
  if (!Array.isArray(middleware)) throw new TypeError('Middleware stack must be an array!')
  for (const fn of middleware) {
    if (typeof fn !== 'function') throw new TypeError('Middleware must be composed of functions!')
  }
  return function (context, next) {
    // last called middleware #
    let index = -1
    return dispatch(0)
    function dispatch (i) {
      if (i <= index) return Promise.reject(new Error('next() called multiple times'))
      index = i
      let fn = middleware[i]
      if (i === middleware.length) fn = next
      if (!fn) return Promise.resolve()
      try {
        return Promise.resolve(fn(context, dispatch.bind(null, i + 1)));
      } catch (err) {
        return Promise.reject(err)
      }
    }
  }
}

```

上面我们说过，我们在使用 use 方法时，传进来的方法会用 middleware 这个方法存起来，然后经过 compose 处理。可以看到，在这个方法中，首先取出第一个方法开始执行，为了实现异步，用 Promise.resolve 包装一下。 
