# The Node.js Event Loop, Timers, and `process.nextTick()`

## What is the Event Loop?

事件循环允许Node.js执行非阻塞I/O操作 - 尽管JavaScript是单线程的 - 只要有可能就将操作卸载到系统内核。

由于大多数现代内核是多线程的，所以它们可以处理在后台执行的多个操作。 当其中一个操作完成时，内核会通知Node.js，以便将适当的回调添加到**poll**队列中以最终执行。 我们稍后将在本主题中进一步详细解释。

## Event Loop Explained

当Node.js启动时，它初始化事件循环；处理提供的输入脚本（或者放入[REPL](https://nodejs.org/api/repl.html#repl_repl)，这在本文档中没有涉及 ），这可能会触发API调用，计划定时器或调用**process.nextTick()**；然后开始处理事件循环。

下图显示了事件循环的操作顺序的简要概述。

```text
   ┌───────────────────────┐
┌─>│        timers         │
│  └──────────┬────────────┘
│  ┌──────────┴────────────┐
│  │     I/O callbacks     │
│  └──────────┬────────────┘
│  ┌──────────┴────────────┐
│  │     idle, prepare     │
│  └──────────┬────────────┘      ┌───────────────┐
│  ┌──────────┴────────────┐      │   incoming:   │
│  │         poll          │<─────┤  connections, │
│  └──────────┬────────────┘      │   data, etc.  │
│  ┌──────────┴────────────┐      └───────────────┘
│  │        check          │
│  └──────────┬────────────┘
│  ┌──────────┴────────────┐
└──┤    close callbacks    │
   └───────────────────────┘
```

> *注意: 每一个方块代表一个event loop的阶段*.

每个阶段都有一个执行回调的FIFO队列。 虽然每个阶段各不相同，但通常当事件循环进入给定阶段时，它将执行特定于该阶段的任何操作，然后执行该阶段队列中的回调，直到队列耗尽或执行到允许的最大回调数目。 当队列耗尽或达到回调限制时，事件循环将移至下一个阶段，依此类推。

由于这些操作中的任何一个都可能调度更多的操作，并且**轮询**阶段中处理的新事件会被内核排队，轮询事件可以在轮询事件正在处理的同时排队。 因此，长时间运行的回调可以使轮询阶段的运行时间远远超过计时器的阈值。请参阅 [timers](https://nodejs.org/en/docs/guides/event-loop-timers-and-nexttick/#timers)和[poll](https://nodejs.org/en/docs/guides/event-loop-timers-and-nexttick/#poll)部分了解更多详情。

> **(*注意:*)** *Windows和Unix / Linux实现之间有一点点的差异，但这对于这个演示并不重要。 最重要的部分在这里。 实际上有七八步，但我们关心的是Node.js实际使用的那些，就是上面的那些*

## Phases Overview

- **timers:** 这个阶段执行被 `setTimeout()` 和 `setInterval()`调度的回调。
- **I/O callbacks:** 执行除了关闭回调，定时器调度的回调，和`setImmediate()`的回调之外的几乎所有回调。
- **idle, prepare:** 仅内部使用。
- **poll:** 检索新的I/O事件；node将在这里适当的阻塞。
- **check:** `setImmediate()` 的回调在这儿被触发。
- **close callbacks:** 例如：socket.on('close', ...).

在事件循环的每次运行之间，Node.js检查是否正在等待任何异步I/O或定时器，如果没有任何异步I/O或定时器，则即刻关闭。

## Phases in Detail

### timers

一个定时器规定了这样的一个**阈值**-*回调在超过多久后会被执行*，而不是一个人们希望何时执行的**精确**时间。定时器的回调会尽可能早的被调度，当到时后。尽管如此，操作系统的调度或者其他正在运行的回调可能会造成定时器回调被延时。

> **(*注意:*)** *技术上来讲, the [poll phase](https://nodejs.org/en/docs/guides/event-loop-timers-and-nexttick/#poll) 控制定时器何时执行*.

举个例子, 设定一个100ms后执行的调度, 然后脚本启动一个异步的文件读操作，这个操作需要95ms:

```javascript
const fs = require('fs');

function someAsyncOperation(callback) {
  // Assume this takes 95ms to complete
  fs.readFile('/path/to/file', callback);
}

const timeoutScheduled = Date.now();

setTimeout(() => {
  const delay = Date.now() - timeoutScheduled;

  console.log(`${delay}ms have passed since I was scheduled`);
}, 100);


// do someAsyncOperation which takes 95 ms to complete
someAsyncOperation(() => {
  const startCallback = Date.now();

  // do something that will take 10ms...
  while (Date.now() - startCallback < 10) {
    // do nothing
  }
});
```

当事件循环进入**poll**阶段时，它有一个空的队列（`fs.readFile（）`还没有完成），所以它将等待剩余的ms数，直到达到最快的定时器的阈值。 当它等待95ms传递时，`fs.readFile（）`完成读取文件，并且需要10ms完成将回调被添加到**poll**队列并执行。 当回调完成时，队列中没有更多的回调，所以事件循环将会看到已经达到最快定时器的阈值，然后回到**定时器**阶段来执行定时器的回调。 在这个例子中，你会看到被调度的定时器和它正在执行的回调之间的总延迟将是105ms。

> 注意: 为了避免**poll**阶段造成事件循环饥饿, [libuv](http://libuv.org)(实现了Node.js事件循环和平台所有异步行为的C库)有个强制上限值 (与系统相关)来停止轮询。

### I/O callbacks

这个阶段执行一些系统操作的回调，例如CP错误等。举个例子，如果TCP scoket在连接时收到了`ECONNREFUSED`错误消息，, 一些 *nix 系统会等待报告这个错误。 这会进入到**I/O callbacks phase**的执行队列。

### poll

**poll**阶段有两个重要的方法:

1. 执行超时的定时器脚本，然后
1. 执行轮询队列中的事件。

当事件循环进入**poll**阶段，并且没有timers调度时，如下两种情况可能发生：

- *如果**(*poll*)**队列**(*不是空的*)***, 事件循环会以同步方式迭代执行对调队列知道队列耗尽，或者达到系统限制。

- *如果**(*poll*)**队列**(*是空的*)***, 如下二者之一会发生:
  - 如果脚本有`setImmediate()`调度, 事件循环会结束**poll**阶段进入到**check**去执行那些调度.
  - 如果脚本**没有**被`setImmediate()`调度, 事件循环会等待直到有回调被添加到队列中，然后立即执行回调。

一旦**poll**队列空了，事件循环会检查所有定时器*看谁到时了*。如果有一个或多个到时, 事件循环会返回到**timers**阶段去执行这些定时器回调。

### check

这个阶段使程序可以在**poll**阶段完成后执行立即执行的回调。如果**poll**阶段空闲下来了并且脚本有`setImmediate()`入队, 事件循环会立即执行check阶段，而不是等待。

`setImmediate()`实际是一个运行在事件循环独立阶段的特殊定时器。它是在poll阶段完成后，用libuv API来实现的回调调度的执行。

通常，当代码执行时，如果需要等待一个进入连接，请求等，事件循环会命中**poll**阶段, 如果一个回调被`setImmediate()`调度并且**poll**阶段是空闲的, **poll**阶段将结束并进入**check**阶段，而不是等待**poll**事件。

### close callbacks

如果一个socket或者句柄意外被关闭(比如`socket.destroy()`), `'close'`事件会在这个阶段发生。 否则它将通过`process.nextTick()`被触发。

## `setImmediate()` vs `setTimeout()`

`setImmediate`和`setTimeout()`是相似的, 但是在被调用的时间上，他们表现出不同的行为方式。

- `setImmediate()`被设计为当前**poll**阶段完成后立即执行脚本。、
- `setTimeout()`当一小段时间阈值到时后，调度一段脚本。

如果两者都是从主模块内部调用的，那么时序将受到进程性能的约束（可能受到机器上运行的其他应用程序的影响）。

例如, 如果我们运行如下没有I/O循环的脚本 (比如：主模块), 两个定时器的执行顺序是不确定的, 这依赖于进程性能:

```javascript
// timeout_vs_immediate.js
setTimeout(() => {
  console.log('timeout');
}, 0);

setImmediate(() => {
  console.log('immediate');
});
```

```shell
$ node timeout_vs_immediate.js
timeout
immediate

$ node timeout_vs_immediate.js
immediate
timeout
```

但是，如果将调用移入到I/O循环中, immediate callback总是会优先执行:

```javascript
// timeout_vs_immediate.js
const fs = require('fs');

fs.readFile(__filename, () => {
  setTimeout(() => {
    console.log('timeout');
  }, 0);
  setImmediate(() => {
    console.log('immediate');
  });
});
```

```shell
$ node timeout_vs_immediate.js
immediate
timeout

$ node timeout_vs_immediate.js
immediate
timeout
```

相比`setTimeout()`使用`setImmediate()`的主要优点是在I/O循环中，他将永远被优先执行，不管有多少定时器到时了。

## process.nextTick()

### 理解process.nextTick()

你也许注意到图中没有`process.nextTick()`，尽管他是异步API。这是因为`process.nextTick()`不是事件循环的技术部分。实际上，`nextTickQueue`会在当前操作完成后被处理，不管当前是在事件循环的哪个阶段。

回头看事件循环图，当你在任何给定的阶段调用`process.nextTick()`，所有传给`process.nextTick()`的回调都会在事件循环进入下一阶段前被执行。这有可能触发一些坏情形，因为**它允许你造成I/O饥饿通过递归调用process.nextTick()**，这会阻止事件循环进入**poll**阶段。

### Why would that be allowed?

为什么类似这样的部分会纳入Node.js？部分由于它的设计哲学是，API应该始终是异步的，即便它不必是。试看如下代码段：

```javascript
function apiCall(arg, callback) {
  if (typeof arg !== 'string')
    return process.nextTick(callback,
                            new TypeError('argument should be string'));
}
```

这个代码块进行参数检查工作，如果参数错误，将错误传递给回调函数。API最近的相关升级允许传递参数给`process.nextTick()`，允许将传入在callback后面的参数作为传递给callback的参数，这样就不必使用函数嵌套。

我们所做的只是将一个错误返回给用户，但是仅仅在我们执行完完用户的其他代码后。通过`process.nextTick()`我们能确保`apiCall()`在执行完余下代码进入事件循环处理前总能执行它的回调。为了实现这个目的，JS的调用栈能被展开去立即执行提供的回调，这回允许用户通过`process.nextTick()`造成递归调用从而无法触发`RangeError:超过了v8的最大调用栈`。

这种哲学可能导致潜在的问题。试看如下代码段：

```javascript
let bar;

// this has an asynchronous signature, but calls callback synchronously
function someAsyncApiCall(callback) { callback(); }

// the callback is called before `someAsyncApiCall` completes.
someAsyncApiCall(() => {
  // since someAsyncApiCall has completed, bar hasn't been assigned any value
  console.log('bar', bar); // undefined
});

bar = 1;
```

用户定义的`someAsncApiCall()`有一个异步申明，但实际上他是同步执行的。`someAsyncApiCall()`被调用时，传递给他的回调函数会在事件循环的相同阶段被执行，因为其实`someAsyncApiCall()`没有做任何别的异步操作。结果，回调函数试图引用的`bar`甚至都不在有效范围内，因为脚本还没有运行完成。

通过将callback放到`process.nextTikc()`中，能使脚本总是可以完成运行，从而保证所有的变量，函数等能先于回调函数的执行完成初始化。阻止事件循环也有可用之处。它便于用户在继续事件循环前弹出错误。这是用`process.nextTick()`改造的前述例子：

```javascript
let bar;

function someAsyncApiCall(callback) {
  process.nextTick(callback);
}

someAsyncApiCall(() => {
  console.log('bar', bar); // 1
});

bar = 1;
```

这是另一个真实的例子：

```javascript
const server = net.createServer(() => {}).listen(8080);

server.on('listening', () => {});
```

一旦传入端口号，端口就会立即被绑定。从而立即出发`listening`的回调函数。可是，`.on('listening')`的回调此时还没有设置好。

绕过这个问题，可以将`'listening'`事件入队到`nextTick()`从而保证脚本完成运行。这可以使用户设置他们想要的事件句柄。

## `process.nextTick()` vs `setImmediate()`

现在我们有两个用户关心的功能相似的调用，但是在命名上有点迷惑人。

- `process.nextTick()` 相同阶段即刻调用。
- `setImmediate()` 在事件循环的下个循环或时序被触发。

功能上来看，名字应该被交换一下。`process.nextTick()` 触发比`setImmediate()`更及时， 但这由于历史原因已经不可能被改变。这个转换将会破坏npm上很大部分的包。每天有很多的模块被增加，意味着每多等一天，更多潜在的包会被破坏。如此，尽管有些迷惑，他们的名字不会交换了。

*我们建议开发者在所有情况下都使用`setImmediate()`，因为更易理解（并且在各种环境下有更好的兼容性，比如browser JS.）*

## Why use process.nextTick()?

有两个主要的理由:

1. 允许用户处理错误, 清除不用的资源, 或者在事件循环继续前尝试再发请求。

1. 在调用栈展开之后继续事件循环之前需要运行回调时，可以派上用场。

符合用户期望的简单例子:

```javascript
const server = net.createServer();
server.on('connection', (conn) => { });

server.listen(8080);
server.on('listening', () => { });
```

虽说`listen()`在事件循环开头执行，但是listening事件的回调位于`setImmediate()`中。除非传入主机名，否则绑定端口会立即发生。为了事件循环能处理，必须命中**poll**阶段，这意味着不可能在listening事件事前接收到触发connection事件的连接。

另一个例子是在构造函数中，声明继承`EventEmitter`并且在构造函数中调用事件：

```javascript
const EventEmitter = require('events');
const util = require('util');

function MyEmitter() {
  EventEmitter.call(this);
  this.emit('event');
}
util.inherits(MyEmitter, EventEmitter);

const myEmitter = new MyEmitter();
myEmitter.on('event', () => {
  console.log('an event occurred!');
});
```

你不能再构造函数中立即激发事件，因为脚本还没有运行到用户为事件设置回调的点。所以，在构造函数中，你可以使用`process.nextTick()`去设置在回调中激发事件，事件将会在构造函数完成后触发，这将实现期望的结果：

```javascript
const EventEmitter = require('events');
const util = require('util');

function MyEmitter() {
  EventEmitter.call(this);

  // use nextTick to emit the event once a handler is assigned
  process.nextTick(() => {
    this.emit('event');
  });
}
util.inherits(MyEmitter, EventEmitter);

const myEmitter = new MyEmitter();
myEmitter.on('event', () => {
  console.log('an event occurred!');
});
```