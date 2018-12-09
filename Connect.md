
# Connect是Express4.x之前版本的核心，为了更好理解Express，需要熟悉Connect。


## 任务： （能出uml图画出核心流程）
### 1. Connect是如何工作的，app.use（即中间件是如何被调用的，这个后续需要跟Koa进行比较）、next工作原理以及一些核心组件原理。
### 2. Express3.x是如何工作的，结果任务1，以及一些核心组件的原理
### 3. Express抛弃了Connect之后是如何架构的。



### 一、Hello Connect 在浏览器输入http://localhost:4000 就能看到Hello from Connect!

    const connect = require('connect');
    const http = require('http');
    const app = connect();

    app.use(function (req, res) {
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

### 二、那么传入http.createServer(app)中的app到底是什么呢？ 答案：function app(req, res, next){}, why?






