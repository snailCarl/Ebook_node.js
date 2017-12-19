# How do I start with Node.js after I installed it?

(如下内容翻译自[官网指导](https://nodejs.org/en/docs/guides/getting-started-guide/))

安装好Node后，开始尝试构建web server。创建名为"app.js"的文件，并粘贴如下代码：

```javascript
const http = require('http');

const hostname = '127.0.0.1';
const port = 3000;

const server = http.createServer((req, res) => {
  res.statusCode = 200;
  res.setHeader('Content-type', 'text/plain');
  res.end('Hello World\n')
});

server.listen(port, hostname, () => {
  console.log(`server running at http://${hostname}:${port}/`);
});
```

完成后，使用`node app.js`运行web server，访问<http://localhost:3000>，将会看到'Hello Wolrd'消息