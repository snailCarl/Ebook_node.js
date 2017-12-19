# Timers in Node.js and beyond

Node.js中的定时器模块提供了设置一段时间后执行代码的功能。使用定时器时不需要通过`require()`导入模块，因为模块的所有方法都是模仿浏览器JavaScript API并且全局可访问的。完整的理解定时器功能会做什么，建议阅读Node.js [Event Loop](https://nodejs.org/en/docs/guides/event-loop-timers-and-nexttick/).

## Controlling the Time Continuum with Node.js

Node.js API提供了好几种到时调度代码的方法。下面的方法有些相似，而且在大多数浏览器中都有，但是Nodel.js提供了这些方法自己的实现。定时器和系统的集成度很高，尽管这些方法是browser API的镜像，在实现上仍有不同。

## "When I say so" Execution ~ *`setTimeout()`*

`setTimeout()` 用户在计划好的毫秒时间后调度代码。这个功能类似于浏览器JavaScript API的[window.setTimeout()](https://developer.mozilla.org/en-US/docs/Web/API/WindowOrWorkerGlobalScope/setTimeout), 不过它不能传递执行连串的代码。

`setTimeout()`接受一个可执行的函数做为第一个参数，一段延时的毫秒作为第二个参数。也可以包含附件参数，如果有，附加参数会传递个最开始的回调函数作为其执行时的参数。例子如下：

```javascript
function myFunc(arg) {
  console.log(`arg was => ${arg}`);
}

setTimeout(myFunc, 1500, 'funky');
```

由于`setTimout()`的调用`myFunc()`会尽肯能接近在1500ms(或1.5s)后执行。

定时器的设置的间隔并不精确。这是因为其他执行的代码可能会阻塞或者挂起时间循环从而退后定时器的运行。唯一的保证是定时器的执行不会早于后面声明的时间间隔。

`setTimeout()`返回一个`Timeout`对象，可以作为被设置的定时器的引用。返回的对象可用于取消定时器(见`clearTimeout()`)，也可以改变行为方式(见`unref()`)。

## "Right after this" Execution ~ *`setImmediate()`*

`setImmediate()`会在当前事件循环的结尾执行代码。代码会在当前事件循环的I/O操作之后，下个事件循环的定时器调度前运行。可以看作“就在此之后”发生，意味着跟在`setImmediate()`后面的代码会在`setImmediate()`函数参数之前执行。

`setImmediate()`的第一个参数是回调函数。后面跟着的参数都会作为回调函数的参数在执行时传递过去。这儿有个例子：

```javascript
console.log('before immediate');

setImmediate((arg) => {
  console.log(`executing immediate: ${arg}`);
}, 'so immediate');

console.log('after immediate');
```

上面传递给`setImmediate()`的函数会在全部代码运行完后执行，输入如下：

```shell
before immediate
after immediate
executing immediate: so immediate
```

`setImmediate()`返回一个`Immediate`对象，能被用于取消immediate调度(参见下面的`clearImmediate()`)。

注意：不要混淆`setImmediate()`和`process.nextTick()`。他们有几个主要的不同。首先`process.nextTick()`会*先于*任何设置的`Immediates`和I/O调度。其次，`process.nextTick()`不能被清除，一旦代码被`process.nextTick()`调度，其执行不能被停止，就像常规执行的函数。参考[这篇指导](https://nodejs.org/en/docs/guides/event-loop-timers-and-nexttick/#process-nexttick)来更深入的理解`process.nextTick()`。

## "Infinite Loop" Execution ~ *`setInterval()`*

如果有一块代码需要执行多次，`setInerval()`能被用于实现该目的。`setInterval()`接收一个函数参数和一个延时的时间参数，然后循环执行无限次数。类似`setTimeout()`，可在延时参数后添加额外的参数，这些附加参数会传递个前面的函数参数。同样，延时也不是精确的，因为操作可能会挂起事件循环，所以只是一个大约的延时。见如下代码：

```javascript
function intervalFunc() {
  console.log('Cant stop me now!');
}

setInterval(intervalFunc, 1500);
```

在上面的代码中，`intervalFunc`将会每隔1500毫秒执行一次，直到它被停止（如下）。

就像`setTimeout()`，`setInterval()`也返回一个`Timeout`对象，能被引用来修改设定的周期调用。

## Clearing the Future

要怎样做，如果一个`Timeout`或者`Immediate`需要被取消？`setTimeout()`，`setImmediate()`，和`setInterval()`返回一个定时器对象能被用来指向设定的`Timeout`或者`Immediate`对象。通过传递前述对象给相关的`clear`函数，执行对象可以停下来。相关函数分别是`clearTimeout()`，`clearImmediate()`和`clearInterval()`。下面有每一个的示例：

```javascript
const timeoutObj = setTimeout(() => {
  console.log('timeout beyond time');
}, 1500);

const immediateObj = setImmediate(() => {
  console.log('immediately executing immediate');
});

const intervalObj = setInterval(() => {
  console.log('interviewing the interval');
}, 500);

clearTimeout(timeoutObj);
clearImmediate(immediateObj);
clearInterval(intervalObj);
```

## Leaving Timeouts Behind

记住`Timeout`对象被`setTimeout`和`setInterval`。`Tiemout`对象提供了两个函数`unref()`和`ref()`来增强`Timeout`的行为。如有有一个设定好的`定时器`调度对象，`unref()`能在那个对象上调用。这回轻微的改变它的行为，不去调用`定时器`回调对象，*如果unref是被最后执行的代码*。在等待执行时，`定时器`对象将不能保活。

同样，调用了`unref()`的定时器对象能通过在通用的对象上调用`ref()`来取消调用效果，以确保它的执行。注意，由于性能原因，它同样不能*精确*恢复初始行为。试看如下示例：

```javascript
const timerObj = setTimeout(() => {
  console.log('will i run?');
});

// if left alone, this statement will keep the above
// timeout from running, since the timeout will be the only
// thing keeping the program from exiting
timerObj.unref();

// we can bring it back to life by calling ref() inside
// an immediate
setImmediate(() => {
  timerObj.ref();
});
```

## Further Down the Event Loop

时间循环的许多内容本篇指导没有覆盖到。要了解更多时间循环机制的奥秘和定制器的操作，检阅这边Node.js的指导：[The Node.js Event Loop, Timers, and process.nextTick()](https://nodejs.org/en/docs/guides/event-loop-timers-and-nexttick/).