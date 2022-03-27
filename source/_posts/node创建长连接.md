---
title: node创建长连接
categories:
  - default
tags:
  - default
date: 2022-02-20 21:12:40
---





通过[request源码](https://github.com/request/request/blob/master/request.js)分析

```javascript
if (!self.agent) {
    if (options.agentOptions) {
      self.agentOptions = options.agentOptions
    }

    if (options.agentClass) {
      self.agentClass = options.agentClass
    } else if (options.forever) {
      var v = version()
      // use ForeverAgent in node 0.10- only
      if (v.major === 0 && v.minor <= 10) {
        self.agentClass = protocol === 'http:' ? ForeverAgent : ForeverAgent.SSL
      } else {
        self.agentClass = self.httpModule.Agent
        self.agentOptions = self.agentOptions || {}
        self.agentOptions.keepAlive = true
      }
    } else {
      self.agentClass = self.httpModule.Agent
    }
  }

  if (self.pool === false) {
    self.agent = false
  } else {
    self.agent = self.agent || self.getNewAgent()
  }
}

  if (self.pool === false) {
    self.agent = false
  } else {
    self.agent = self.agent || self.getNewAgent()
  }

```

通过源码可以看到，要使用node创建一个长连接的http请求，有三种方式：

1. 直接塞给options已给 agent。
2. 给options一个agentClass
3. options.forever的值设置为true

但是，如果追踪代码的话，最后发现，基本上都是去new 一个 Agent，这里的pool，其实就是globalPool，

```javascript
var globalPool = {}

debug(options)
  if (!self.pool && self.pool !== false) {
    self.pool = globalPool
  }
```

那么，看看Agent的源码：他在node的[http模块](https://github.com/nodejs/node/blob/v17.0.0/lib/_http_client.js)中 

```javascript
function Agent(options) {
  if (!(this instanceof Agent))
    return new Agent(options);

  FunctionPrototypeCall(EventEmitter, this);

  this.defaultPort = 80;
  this.protocol = 'http:';

  this.options = { __proto__: null, ...options };

  // Don't confuse net and make it think that we're connecting to a pipe
  this.options.path = null;
  this.requests = ObjectCreate(null);
  this.sockets = ObjectCreate(null);
  this.freeSockets = ObjectCreate(null);
  this.keepAliveMsecs = this.options.keepAliveMsecs || 1000;
  this.keepAlive = this.options.keepAlive || false;
  this.maxSockets = this.options.maxSockets || Agent.defaultMaxSockets;
  this.maxFreeSockets = this.options.maxFreeSockets || 256;
  this.scheduling = this.options.scheduling || 'lifo';
  this.maxTotalSockets = this.options.maxTotalSockets;
  this.totalSocketCount = 0;

  validateOneOf(this.scheduling, 'scheduling', ['fifo', 'lifo']);

  if (this.maxTotalSockets !== undefined) {
    validateNumber(this.maxTotalSockets, 'maxTotalSockets');
    if (this.maxTotalSockets <= 0 || NumberIsNaN(this.maxTotalSockets))
      throw new ERR_OUT_OF_RANGE('maxTotalSockets', '> 0',
                                 this.maxTotalSockets);
  } else {
    this.maxTotalSockets = Infinity;
  }

  this.on('free', (socket, options) => {
    const name = this.getName(options);
    debug('agent.on(free)', name);

    // TODO(ronag): socket.destroy(err) might have been called
    // before coming here and have an 'error' scheduled. In the
    // case of socket.destroy() below this 'error' has no handler
    // and could cause unhandled exception.

    if (!socket.writable) {
      socket.destroy();
      return;
    }

    const requests = this.requests[name];
    if (requests && requests.length) {
      const req = ArrayPrototypeShift(requests);
      const reqAsyncRes = req[kRequestAsyncResource];
      if (reqAsyncRes) {
        // Run request within the original async context.
        reqAsyncRes.runInAsyncScope(() => {
          asyncResetHandle(socket);
          setRequestSocket(this, req, socket);
        });
        req[kRequestAsyncResource] = null;
      } else {
        setRequestSocket(this, req, socket);
      }
      if (requests.length === 0) {
        delete this.requests[name];
      }
      return;
    }

    // If there are no pending requests, then put it in
    // the freeSockets pool, but only if we're allowed to do so.
    const req = socket._httpMessage;
    if (!req || !req.shouldKeepAlive || !this.keepAlive) {
      socket.destroy();
      return;
    }

    const freeSockets = this.freeSockets[name] || [];
    const freeLen = freeSockets.length;
    let count = freeLen;
    if (this.sockets[name])
      count += this.sockets[name].length;

    if (this.totalSocketCount > this.maxTotalSockets ||
        count > this.maxSockets ||
        freeLen >= this.maxFreeSockets ||
        !this.keepSocketAlive(socket)) {
      socket.destroy();
      return;
    }

    this.freeSockets[name] = freeSockets;
    socket[async_id_symbol] = -1;
    socket._httpMessage = null;
    this.removeSocket(socket, options);

    socket.once('error', freeSocketErrorListener);
    ArrayPrototypePush(freeSockets, socket);
  });

  // Don't emit keylog events unless there is a listener for them.
  this.on('newListener', maybeEnableKeylog);
}
```

注意这句话

```javascript
 // If there are no pending requests, then put it in
    // the freeSockets pool, but only if we're allowed to do so.
const req = socket._httpMessage;
if (!req || !req.shouldKeepAlive || !this.keepAlive) {
      socket.destroy();
      return;
    }
```

如果不是keepAlive的，一个请求结束，这个socket将果断断开，但其实还有一个条件，那就是，已经没有等待要处理的请求了，也是直接断开，避免浪费内存。



```javascript
function tickOnSocket(req, socket) {
  const parser = parsers.alloc();
  req.socket = socket;
  const lenient = req.insecureHTTPParser === undefined ?
    isLenient() : req.insecureHTTPParser;
  parser.initialize(HTTPParser.RESPONSE,
                    new HTTPClientAsyncResource('HTTPINCOMINGMESSAGE', req),
                    req.maxHeaderSize || 0,
                    lenient ? kLenientAll : kLenientNone,
                    0);
  parser.socket = socket;
  parser.outgoing = req;
  req.parser = parser;

  socket.parser = parser;
  socket._httpMessage = req;//是在这赋值的。

```



```javascript
ClientRequest.prototype.onSocket = function onSocket(socket, err) {
  // TODO(ronag): Between here and onSocketNT the socket
  // has no 'error' handler.
  process.nextTick(onSocketNT, this, socket, err);
};

function onSocketNT(req, socket, err) {
  if (req.destroyed || err) {
    req.destroyed = true;

    function _destroy(req, err) {
      if (!req.aborted && !err) {
        err = connResetException('socket hang up');
      }
      if (err) {
        req.emit('error', err);
      }
      req._closed = true;
      req.emit('close');
    }

    if (socket) {
      if (!err && req.agent && !socket.destroyed) {
        socket.emit('free');
      } else {
        finished(socket.destroy(err || req[kError]), (er) => {
          if (er?.code === 'ERR_STREAM_PREMATURE_CLOSE') {
            er = null;
          }
          _destroy(req, er || err);
        });
        return;
      }
    }

    _destroy(req, err || req[kError]);
  } else {
    tickOnSocket(req, socket);
    req._flush();
  }
}
```



这是底层Socket的setKeepAlive，

```javascript
Socket.prototype.setKeepAlive = function(setting, msecs) {
  if (!this._handle) {
    this.once('connect', () => this.setKeepAlive(setting, msecs));
    return this;
  }

  if (this._handle.setKeepAlive)
    this._handle.setKeepAlive(setting, ~~(msecs / 1000));

  return this;
};
```



这个是SetKeepAlive的底层C++代码

```c++
void TCPWrap::SetKeepAlive(const FunctionCallbackInfo<Value>& args) {
  TCPWrap* wrap;
  ASSIGN_OR_RETURN_UNWRAP(&wrap,
                          args.Holder(),
                          args.GetReturnValue().Set(UV_EBADF));
  Environment* env = wrap->env();
  int enable;
  if (!args[0]->Int32Value(env->context()).To(&enable)) return;
  unsigned int delay = static_cast<unsigned int>(args[1].As<Uint32>()->Value());
  int err = uv_tcp_keepalive(&wrap->handle_, enable, delay);
  args.GetReturnValue().Set(err);
}

```

```c++
void TCPWrap::Initialize(Local<Object> target,
                         Local<Value> unused,
                         Local<Context> context,
                         void* priv) {
  Environment* env = Environment::GetCurrent(context);

  Local<FunctionTemplate> t = env->NewFunctionTemplate(New);
  t->InstanceTemplate()->SetInternalFieldCount(StreamBase::kInternalFieldCount);

  // Init properties
  t->InstanceTemplate()->Set(FIXED_ONE_BYTE_STRING(env->isolate(), "reading"),
                             Boolean::New(env->isolate(), false));
  t->InstanceTemplate()->Set(env->owner_symbol(), Null(env->isolate()));
  t->InstanceTemplate()->Set(env->onconnection_string(), Null(env->isolate()));

  t->Inherit(LibuvStreamWrap::GetConstructorTemplate(env));

  env->SetProtoMethod(t, "open", Open);
  env->SetProtoMethod(t, "bind", Bind);
  env->SetProtoMethod(t, "listen", Listen);
  env->SetProtoMethod(t, "connect", Connect);
  env->SetProtoMethod(t, "bind6", Bind6);
  env->SetProtoMethod(t, "connect6", Connect6);
  env->SetProtoMethod(t, "getsockname",
                      GetSockOrPeerName<TCPWrap, uv_tcp_getsockname>);
  env->SetProtoMethod(t, "getpeername",
                      GetSockOrPeerName<TCPWrap, uv_tcp_getpeername>);
  env->SetProtoMethod(t, "setNoDelay", SetNoDelay);
  env->SetProtoMethod(t, "setKeepAlive", SetKeepAlive);//这里注册
```



```javascript

In the case of a premature connection close before the response is received,
the following events will be emitted in the following order:

* `'socket'`
* `'error'` with an error with message `'Error: socket hang up'` and code
  `'ECONNRESET'`
* `'close'`

If `req.destroy()` is called before a socket is assigned, the following
events will be emitted in the following order:

* (`req.destroy()` called here)
* `'error'` with an error with message `'Error: socket hang up'` and code
  `'ECONNRESET'`
* `'close'`

If `req.destroy()` is called before the connection succeeds, the following
events will be emitted in the following order:

* `'socket'`
* (`req.destroy()` called here)
* `'error'` with an error with message `'Error: socket hang up'` and code
  `'ECONNRESET'`
* `'close'`


```



dns解析

```javascript
function lookupAndListen(self, port, address, backlog, exclusive, flags) {
  if (dns === undefined) dns = require('dns');
  dns.lookup(address, function doListen(err, ip, addressType) {
    if (err) {
      self.emit('error', err);
    } else {
      addressType = ip ? addressType : 4;
      listenInCluster(self, ip, port, addressType,
                      backlog, undefined, exclusive, flags);
    }
  });
}

```



创建socket，是管道，或者 tcp

```javascript
function createHandle(fd, is_server) {
  validateInt32(fd, 'fd', 0);
  const type = guessHandleType(fd);
  if (type === 'PIPE') {
    return new Pipe(
      is_server ? PipeConstants.SERVER : PipeConstants.SOCKET
    );
  }

  if (type === 'TCP') {
    return new TCP(
      is_server ? TCPConstants.SERVER : TCPConstants.SOCKET
    );
  }

  throw new ERR_INVALID_FD_TYPE(type);
}
```



老外的测试用例

```javascript
tape('keepAlive is timed', function (t) {
  var agent = new http.Agent({ keepAlive: true })
  var options = { time: true, agent: agent }
  var start1 = new Date().getTime()

  request('http://localhost:' + plainServer.port + '/', options, function (err1, res1, body1) {
    var end1 = new Date().getTime()

    // ensure the first request's timestamps look ok
    t.equal((res1.timingStart >= start1), true)
    t.equal((start1 <= end1), true)

    t.equal((res1.timings.socket >= 0), true)
    t.equal((res1.timings.lookup >= res1.timings.socket), true)
    t.equal((res1.timings.connect >= res1.timings.lookup), true)
    t.equal((res1.timings.response >= res1.timings.connect), true)

    // open a second request with the same agent so we re-use the same connection
    var start2 = new Date().getTime()
    request('http://localhost:' + plainServer.port + '/', options, function (err2, res2, body2) {
      var end2 = new Date().getTime()

      // ensure the second request's timestamps look ok
      t.equal((res2.timingStart >= start2), true)
      t.equal((start2 <= end2), true)

      // ensure socket==lookup==connect for the second request
      t.equal((res2.timings.socket >= 0), true)
      t.equal((res2.timings.lookup === res2.timings.socket), true)
      t.equal((res2.timings.connect === res2.timings.lookup), true)
      t.equal((res2.timings.response >= res2.timings.connect), true)

      // explicitly shut down the agent
      if (typeof agent.destroy === 'function') {
        agent.destroy()
      } else {
        // node < 0.12
        Object.keys(agent.sockets).forEach(function (name) {
          agent.sockets[name].forEach(function (socket) {
            socket.end()
          })
        })
      }

      t.end()
    })
  })
})
```

