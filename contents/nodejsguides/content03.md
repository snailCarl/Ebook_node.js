# Overview of Blocking vs Non-Blocking

本概述讲解Node.js中阻塞调用和非阻塞调用的区别。涉及到了时间循环和libuv相关知识，但并要求事先了解这些主题。读者需要具备JavaScript语言和Node.js回调模式的相关理解。

  "I/O" 主要是指基于[libuv](http://libuv.org)库的各种和系统磁盘以及网络的交互。

## Blocking

**Blocking** 当附加的JavaScript在Node.js进程中执行时，必须等到非JavaScript的操作执行完成。这是因为事件循环不能在**阻塞**操作发生时继续运行JavaScript。

在Node.js中，由于CPU密集而不是等待非JavaScript操作（例如I / O）而表现出糟糕的性能的JavaScript通常不被称为**阻塞**。在Node.js的标准库中使用libuv的同步方法是最常用的**阻塞**操作。 原生模块也有**阻塞**方法。

在Node.js标准库中，所有的I/O方法都提供了异步的版本，也就是所谓的**非阻塞**，然后接受一个回调函数作为参数。一些方法还会存在对应的**阻塞**版本，方法的名称对应以`Sync`结尾。

## Comparing Code

**阻塞**方法同步执行，**非阻塞**方法异步执行。

以文件模块为例，下面是一个**同步**的文件读操作：

```javascript
const fs = require('fs');
const data = fs.readFileSync('/file.md'); // blocks here until file is read
```

这里是一个对应的**异步**例子：

```javascript
const fs = require('fs');
fs.readFile('/file.md', (err, data) => {
  if (err) throw err;
});
```

第一个例子似乎更简单，但是确定在于第二行的**阻塞**操作。其他另外的JavaScript必须等到文件全部读完才能继续执行。值得注意的是，在同步版本中如果有错误被抛出，那么它必须被捕获，否则进程崩溃。在异步版本中，取决于作者是否需要抛出错误使之可见。

让我们稍微扩展一下这个例子：

```javascript
const fs = require('fs');
const data = fs.readFileSync('/file.md'); // blocks here until file is read
console.log(data);
// moreWork(); will run after console.log
```

这儿是个相似的，不完全对应的异步例子：

```javascript
const fs = require('fs');
fs.readFile('/file.md', (err, data) => {
  if (err) throw err;
  console.log(data);
});
// moreWork(); will run before console.log
```

在上面的第一个例子中，`console.log`会在`moreWork()`之前执行. 在第第二个例子中`fs.readFile()`是 **非阻塞**的， 所以Javascript可以继续执行，`moreWork()`会先被调用。这种允许`moreWork()`无需等待文件读操作完成而先执行的能力是允许更高吞吐量的关键设计选择。

## Concurrency and Throughput（并发和吞吐量）

Node.js中的JavaScript执行是单线程的，所以并发是指事件循环完成其他工作后执行JavaScript回调函数的能力。 任何期望以并发方式运行的代码都必须允许事件循环在非JavaScript操作（如I / O）正在发生时继续运行。

举一个例子，让我们考虑一个情况，每个Web服务器的请求需要50ms才能完成，而50ms中的45ms是可以异步完成的数据库I/O。 选择**非阻塞**异步操作可以释放每个请求45ms以处理其他请求。 这只是通过选择使用**非阻塞**方法而不是**阻塞**方法，在处理容量上的显着差异。

事件循环与许多其他语言中的模型不同，在这些语言中可以创建额外的线程来处理并发工作。

## Dangers of Mixing Blocking and Non-Blocking Code

处理I/O时应该避免一些模式。 我们来看一个例子：

```javascript
const fs = require('fs');
fs.readFile('/file.md', (err, data) => {
  if (err) throw err;
  console.log(data);
});
fs.unlinkSync('/file.md');
```

在上面的例子中，`fs.unlinkSync（）`很可能在`fs.readFile（）`之前运行，它会在实际读取之前删除`file.md`。 一个更好的实现完全**非阻塞**的方式，并保证正确执行顺序的例子如下：

```javascript
const fs = require('fs');
fs.readFile('/file.md', (readFileErr, data) => {
  if (readFileErr) throw readFileErr;
  console.log(data);
  fs.unlink('/file.md', (unlinkErr) => {
    if (unlinkErr) throw unlinkErr;
  });
});
```

上面的例子将**非阻塞**的`fs.unlink()`调用放到了`fs.readFile()`回调函数的内部，从而保证了正确的操作顺序。

## Additional Resources

- [libuv](http://libuv.org)
- [About Node.js](https://nodejs.org/en/about/)