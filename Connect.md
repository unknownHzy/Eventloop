
# Connect是Express4.x之前版本的核心，为了更好理解Express，需要熟悉Connect。


## 任务： （能出uml图画出核心流程）
### 1. Connect是如何工作的，app.use（即中间件是如何被调用的，这个后续需要跟Koa进行比较）、next工作原理以及一些核心组件原理。
### 2. Express3.x是如何工作的，结果任务1，以及一些核心组件的原理
### 3. Express抛弃了Connect之后是如何架构的。



### 一、Hello Connect 在浏览器输入http://localhost:4000 就能看到Hello from Connect!

    const connect = require('connect');
    const http = require('http');
    const app = connect();

    app.use('/', function (req, res) {
        res.end('Hello from Connect!');
    });

    http.createServer(app).listen(4000);

#### 1）http.createServer(requestListener) 返回一个httpServer实例，将requestListener函数绑定到'request'事件中
https://nodejs.org/dist/latest-v10.x/docs/api/http.html#http_http_createserver_options_requestlistener
   
    exports.createServer = function(requestListener) {
      return new Server(requestListener);
      
      function Server(requestListener) {
          if (requestListener) {
            util.inherits(Server, EventEmitter);
            this.addListener('request', requestListener);//EventEmitter.prototype.on = EventEmitter.prototype.addListener;
          }
      }
    };
    
**问题1：什么时候触发这个request事件呢？
在_http_server.js中的**

    function connectionListener(socket) {
      ...
      if (req.headers.expect !== undefined &&
        (req.httpVersionMajor == 1 && req.httpVersionMinor == 1) &&
        continueExpression.test(req.headers['expect'])) {
        res._expect_continue = true;
        if (EventEmitter.listenerCount(self, 'checkContinue') > 0) {
          self.emit('checkContinue', req, res);
        } else {
          res.writeContinue();
          self.emit('request', req, res);   //here 就是在触发connect事件，调用其callback的时候，触发的request事件
        }
      } else {
        self.emit('request', req, res);     //here 就是在触发connect事件，调用其callback的时候，触发的request事件
      }
      ...
    }
**问题2： 在self.emit('request', req, res); 触发request的时候，向requestListener传递了两个参数：http的req与res
所以requestListener至少得是function(req, res, ...) {}

### 二、const app = connect(); 做了什么
    module.exports = createServer;
    
    //app继承了proto（包含了use/handle/listen重要方法与EventEmitter中原型链中的方法，浅拷贝）
    function createServer() {
      function app(req, res, next){ app.handle(req, res, next); }
      merge(app, proto);
      merge(app, EventEmitter.prototype);
      app.route = '/';
      app.stack = [];
      return app;
    }

### 二、app.use()做了什么？ 答案是： 往app的中间件栈中不断添加{route: route, handle: fn}
    proto.use = function use(route, fn) {
      var handle = fn;
      var path = route;

      // default route to '/', 若直接使用app.use(function(req, res) {});则默认的route是 '/'
      if (typeof route !== 'string') { 
        handle = route;
        path = '/';
      }

      // wrap sub-apps
      ....

      // wrap vanilla http.Servers
      ....

      // strip trailing slash .将路径最后面的'/'截掉，若存在的话
      if (path[path.length - 1] === '/') {
        path = path.slice(0, -1);
      }

      // add the middleware，在createServer中初始化了app.stack = [];
      debug('use %s %s', path || '/', handle.name || 'anonymous');
      this.stack.push({ route: path, handle: handle }); 

      return this;
    };


### 三、那么传入http.createServer(app)中的app到底是什么呢？ 答案：function app(req, res, next){}, why?

    const connect = require('connect');
    const app = connect();
    
    module.exports = createServer;
    var proto = {};
    function createServer() {
      function app(req, res, next){ app.handle(req, res, next); }
      merge(app, proto); 
      merge(app, EventEmitter.prototype);
      app.route = '/';
      app.stack = [];
      return app;// app是app.handle的别名，那哪里来的handle方法呢？（app是function，被return出去了，就是说handle方法这里还没有被执行）
    }
    
    proto.handle = function handle(req, res, out) {
      var index = 0;
      var protohost = getProtohost(req.url) || '';
      var removed = '';
      var slashAdded = false;
      var stack = this.stack;

      // final function handler
      var done = out || finalhandler(req, res, {
        env: env,
        onerror: logerror
      });

      // store the original URL
      req.originalUrl = req.originalUrl || req.url;

      function next(err) {
        if (slashAdded) {
          req.url = req.url.substr(1);
          slashAdded = false;
        }

        if (removed.length !== 0) {
          req.url = protohost + removed + req.url.substr(protohost.length);
          removed = '';
        }

        // next callback
        var layer = stack[index++];

        // all done
        if (!layer) {
          defer(done, err);
          return;
        }

        // route data
        var path = parseUrl(req).pathname || '/';
        var route = layer.route;

        // skip this layer if the route doesn't match
        if (path.toLowerCase().substr(0, route.length) !== route.toLowerCase()) {
          return next(err);
        }

        // skip if route match does not border "/", ".", or end
        var c = path.length > route.length && path[route.length];
        if (c && c !== '/' && c !== '.') {
          return next(err);
        }

        // trim off the part of the url that matches the route
        if (route.length !== 0 && route !== '/') {
          removed = route;
          req.url = protohost + req.url.substr(protohost.length + removed.length);

          // ensure leading slash
          if (!protohost && req.url[0] !== '/') {
            req.url = '/' + req.url;
            slashAdded = true;
          }
        }

        // call the layer handle
        call(layer.handle, route, err, req, res, next);
          }

          next();
        };







