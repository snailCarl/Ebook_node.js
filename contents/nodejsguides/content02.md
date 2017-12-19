# Anatomy of an HTTP Transaction

本指南的目的是为了对Http的处理过程有一个很好的了解。我们假定读者对于Http request工作机制有一个大致了解，而不关心编程语言环境。我们也假定读者对Node.js的[`EventEmitters`](https://nodejs.org/api/events.html)和[`Streams`](https://nodejs.org/api/stream.html)也有所知晓。如果你不是很熟悉它们，最好快速浏览一下API docs中的相关内容。

## Create the Server

任何node web server 应用程序都会以创建web server对象为开始。这项工作通过调用[`createServer`](https://nodejs.org/api/http.html#http_http_createserver_requestlistener)来完成。

```javascript
const http = require('http');

const server = http.createServer((request, response) => {
  // magic happens here!
});
```

每当发生针对该服务的请求，传入[`createServer`](https://nodejs.org/api/http.html#http_http_createserver_requestlistener)的函数就会被调用，所以它也被称作request句柄。事实上[`createServer`](https://nodejs.org/api/http.html#http_http_createserver_requestlistener)返回的[`Server`](https://nodejs.org/api/http.html#http_class_http_server)对象是一个[`EventEmitter`](https://nodejs.org/api/events.html#events_class_eventemitter)，而我们这里所做的不过是创建一个`server`对象，然后添加监听器的简写。

```javascript
const server = http.createServer();
server.on('request', (request, response) => {
  // the same kind of magic happens here!
});
```

当http请求命中服务时，node调用request句柄函数并封装好`transcation`，`request`和`response`对象以方便处理。稍后即将看到。

为了实际服务request，需要在serve对象上调用[`listen`](https://nodejs.org/api/http.html#http_server_listen_port_hostname_backlog_callback)方法。大多数情况下，你需要做的只是将你期望服务监听在什么端口上的端口号传给`listen`。其他选项，可参考[`API reference`](https://nodejs.org/api/http.html)。

## Method, URL and Headers

当处理一个请求，你首先想做的可能是查看method和URL，以便执行合适的动作。通过在`request`对象上封装一些便于处理的属性，Node让这个过程变的相对简单。

```javascript
const { method, url } = request;
```

> 注意：`request`对象是一个`IncomingMessage`的实例

这里的`method`是一个通常的HTTP method/动词。`url`是指完整URL去掉服务器，协议和端口后的剩余部分，这意味着包括第三个斜杠后的所有部分。

谈到Headers了，在`request`中他们封装在叫`headers`的对象中。

```javascript
const { headers } = request;
const userAgent = headers['user-agent'];
```

需要重点指出的是，不管客户端实际上如何发送他们，所有的headers一律用小写字母表示。某种程度上，这简化了解析headers的任务。

如果一些headers重复了，他们的值是被覆盖还是连接成用逗号分隔的字串，取决于header的设置。有些情形，这会产生一些困扰，所以可以使用[`rawHeaders`](https://nodejs.org/api/http.html#http_message_rawheaders)。

## Request Body

当收到一个`post`或者`get`请求，请求体对程序而言是重要的。获取包体数据比访问heades稍微复杂一点。传递给句柄的`request`对象实现了[`ReadableStream`](https://nodejs.org/api/stream.html#stream_class_stream_readable)接口。这stream可以被监听或者piped到别处，就像其他流一样。通过监听流的`'data'`和`'end'`事件，我们可以从流中抓出数据。

发射到每个`data`事件中的数据块是一个[`Buffer`](https://nodejs.org/api/buffer.html)。如果你事先知道它是字符串数据，你可以将数据收集到数组，然后在`end`事件中，连接并且将它字符串化。

```javascript
let body = [];
request.on('data', (chunk) => {
  body.push(chunk);
}).on('end', () => {
  body = Buffer.concat(body).toString();
  // at this point, `body` has the entire request body stored in it as a string
});
```

> 大多数时候这些处理很单调。幸运的是，在[`npm`](https://www.npmjs.com/)上有一些模块，例如[`concat-stream`](https://www.npmjs.com/package/concat-stream)和[`body`](https://www.npmjs.com/package/body)能帮我们隐藏这些逻辑细节。不过继续之前，了解这些原理是重要的。这也就是你看到这些的原因。

## A Quick Thing About Errors

`request`对象是一个[`ReadableStream`](https://nodejs.org/api/stream.html#stream_class_stream_readable)，它同样也是一个[`EventEmitter`](https://nodejs.org/api/events.html#events_class_eventemitter)，并且当发生错误时表现出EventEmitter行为。

在`request`流中，一个错误通过在流上发射`'error'`事件来表现自身。**如果你没有监听error事件，错误将被抛出，并会引起Node.js程序崩溃。**因此你应该在request流上增加`'error'`监听器，哪怕你仅在监听器中简单的打印错误然后继续。(话虽如此，最好是发送相应的HTTP error响应。稍后可见。)

```javascript
request.on('error', (err) => {
  // This prints the error message and stack trace to `stderr`.
  console.error(err.stack);
});
```

还有其他的错误处理方式，比如其他的抽象化和工具。但要铭记，错误会发生，你必须处理他们。

## What We've Got so Far

到目前为止，我们了解了如何创建一个服务，如何从请求中提取method，URL，headers和body。总结一下，它看起来将像下面这样：

```javascript
const http = require('http');

http.createServer((request, response) => {
  const { headers, method, url } = request;
  let body = [];
  request.on('error', (err) => {
    console.error(err);
  }).on('data', (chunk) => {
    body.push(chunk);
  }).on('end', () => {
    body = Buffer.concat(body).toString();
    // At this point, we have the headers, method, url and body, and can now
    // do whatever we need to in order to respond to this request.
  });
}).listen(8080); // Activates this server, listening on port 8080.
```

当运行这个例子，我们能收到请求，但是没有响应给它。事实上，如果在浏览器中触发样例，你的请求将会超时，因为没有任何响应发向客户端。

截止现在，我们根本还没有触碰到`response`对象，它既是一个[`ServerResponse`](https://nodejs.org/api/http.html#http_class_http_serverresponse)实例，也是一个[`WritableStream`](https://nodejs.org/api/stream.html#stream_class_stream_writable)。它包含许多向客户端回发数据的有用工具。我们马上了解它。

## HTTP Status Code

如果你不想费心设置它，那么一个response的HTTP status code将默认为200。当然，不是每一个response都保证如此，而且有时你可能明确的想发送一个不同的状态码。那么，请设置`statusCode`属性。

```javascript
response.statusCode = 404; // Tell the client that the resource wasn't found.
```

稍后可见，有一些做这些的其他快捷方式。

## Setting Response Headers

通过[`setHeader`](https://nodejs.org/api/http.html#http_response_setheader_name_value)方法可以方便的设置Headers。

```javascript
response.setHeader('Content-Type', 'application/json');
response.setHeader('X-Powered-By', 'bacon');
```

当设置response的headers时，大小写是敏感的。如果你重复设置，最后的值其作用。

## Explicitly Sending Header Data

我们已经讨论了设置headers和状态码的方法，假定你使用"implicit headers"（隐式的头）。这意味着你指望在发送body data前由node适时的去发送headers。

如果你想这样，你可以显示的将headers写到response stream中。调用[`writeHead`](https://nodejs.org/api/http.html#http_response_writehead_statuscode_statusmessage_headers)即可，它将写入状态码和headers到流中。

```javascript
response.writeHead(200, {
  'Content-Type': 'application/json',
  'X-Powered-By': 'bacon'
});
```

一旦你设置了头数据（不管显示还是隐式），你就准备好可以发送response数据了。

## Sending Response Body

由于`response`对象是一个[`WritableStream`](https://nodejs.org/api/stream.html#stream_class_stream_writable)，写入响应包体数据给客户端，变成了仅仅是调用通常的流方法的问题。

```javascript
response.write('<html>');
response.write('<body>');
response.write('<h1>Hello, World!</h1>');
response.write('</body>');
response.write('</html>');
response.end();
```

`end`方法也能向流的末尾输入一些选项数据，所以我们可以如下简化上面的例子。

```javascript
response.end('<html><body><h1>Hello, World!</h1></body></html>');
```

> 注意：在向body写入数据块前，设置状态和头是重要的。意义在于，在HTTP响应中headers先于body到达。

## Another Quick Thing About Errors

`response`流也能发射`'error'`事件，一些情形下，你同样必须处理它。所有对于`request`流的错误处理建议，同样适用于这儿。

## Put It All Together

现在我们已经学习了如何生成HTTP应答，下面来一个完整的流程。基于早前构建着的样例，我们将创建一个服务，该服务将用户发给我们的数据原样送还给用户。我们使用`JSON.stringfy`将数据格式化为JSON字串。

```javascript
const http = require('http');

http.createServer((request, response) => {
  const { headers, method, url } = request;
  let body = [];
  request.on('error', (err) => {
    console.error(err);
  }).on('data', (chunk) => {
    body.push(chunk);
  }).on('end', () => {
    body = Buffer.concat(body).toString();
    // BEGINNING OF NEW STUFF

    response.on('error', (err) => {
      console.error(err);
    });

    response.statusCode = 200;
    response.setHeader('Content-Type', 'application/json');
    // Note: the 2 lines above could be replaced with this next one:
    // response.writeHead(200, {'Content-Type': 'application/json'})

    const responseBody = { headers, method, url, body };

    response.write(JSON.stringify(responseBody));
    response.end();
    // Note: the 2 lines above could be replaced with this next one:
    // response.end(JSON.stringify(responseBody))

    // END OF NEW STUFF
  });
}).listen(8080);
```

## Echo Server Example

让我们将前面的样例简化为一个简单的回声服务。这个服务只需将从request中接收到的任何数据，发送到response中。需要我们做的全部工作，只是从request流中抓取数据并写入到response流中，就像我们前面所做的。

```javascript
const http = require('http');

http.createServer((request, response) => {
  let body = [];
  request.on('data', (chunk) => {
    body.push(chunk);
  }).on('end', () => {
    body = Buffer.concat(body).toString();
    response.end(body);
  });
}).listen(8080);
```

现在，让我们调整一下。我们仅仅想发送一个回应，在如下情况下：

- 请求的方法是 `GET`.
- URL是`/echo`.

其他情形，我们简单应答为404

```javascript
const http = require('http');

http.createServer((request, response) => {
  if (request.method === 'GET' && request.url === '/echo') {
    let body = [];
    request.on('data', (chunk) => {
      body.push(chunk);
    }).on('end', () => {
      body = Buffer.concat(body).toString();
      response.end(body);
    });
  } else {
    response.statusCode = 404;
    response.end();
  }
}).listen(8080);
```

> 注意: 通过这种方式检查URL，我们在做一种“路由”。 其他形式的路由可以简单到一个`switch`声明或者复杂到一个类似[`express`](https://www.npmjs.com/package/express)整个框架。 如果你只是通过路由寻找而没有其他事情, 试试[`router`](https://www.npmjs.com/package/router).

很不错! 现在让我们来做一点简化。 记住， `request`对象是一个[`ReadableStream`](https://nodejs.org/api/stream.html#stream_class_stream_readable) ，`response`对象是一个[`WritableStream`](https://nodejs.org/api/stream.html#stream_class_stream_writable)。这意味着我们能使用[`pipe`](https://nodejs.org/api/stream.html#stream_readable_pipe_destination_options)直接将数据从一个导入另一个。这正是我们想让回声服务做的事情。

```javascript
const http = require('http');

http.createServer((request, response) => {
  if (request.method === 'GET' && request.url === '/echo') {
    request.pipe(response);
  } else {
    response.statusCode = 404;
    response.end();
  }
}).listen(8080);
```

对，这就是流！

不过还没有全部完成。如前多次提及，错误可能并且总是会发生，我们需要进行错误处理。

处理request流的错误，我们将错误打印到`stderr(标准错误输出)`中然后发送400的状态码，表明这是一个`Bad Request`。在实际应用程序中，我们会调查错误原因并且确定正确的状态码和消息。对于常见错误，请参考[`Error` documentation](https://nodejs.org/api/errors.html).

对于reponse错误，我们仅但输出到`stdout(标准输出)`。

```javascript
const http = require('http');

http.createServer((request, response) => {
  request.on('error', (err) => {
    console.error(err);
    response.statusCode = 400;
    response.end();
  });
  response.on('error', (err) => {
    console.error(err);
  });
  if (request.method === 'GET' && request.url === '/echo') {
    request.pipe(response);
  } else {
    response.statusCode = 404;
    response.end();
  }
}).listen(8080);
```

现在我们已经掌握了处理HTTP请求的大部分基础知识。到此，你已具备如下技能：

- 实例化一个HTTP服务，为这个服务注册一个请求句柄函数，让服务监听某个端口。
- 从`request`对象中获取头，URL，method和包体数据。
- 基于URL或者`request`对象中的其他数据，做出路由选择。
- 通过`response`对象发送headers, HTTP status codes和包体数据。
- 通过管道命令将数据从`request`对象输入到`response`对象。
- 处理`request`流`response`流的错误。

基于这些基础，可以构建许多典型用例的Node.js HTTP服务器。这些Node.js API还能提供其他许多功能，请确保阅读一下相关API文档：[`EventEmitters`](https://nodejs.org/api/events.html), [`Streams`](https://nodejs.org/api/stream.html), and [`HTTP`](https://nodejs.org/api/http.html).