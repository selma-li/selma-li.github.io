---
layout: post
title: 理解浏览器与Nodejs中的event loop
date: 2019-01-26 20:18:03
tags: [Koa, NodeJs]
categories:
 - JavaScript
---

# event loop是什么
event loop即事件循环，是指浏览器或Node的一种解决javaScript单线程运行时不会阻塞的一种机制，也就是我们经常使用异步的原理。
<!-- more -->

# 浏览器中的事件循环
## 主线程、执行栈、任务队列
JavaScript 有一个 main thread __主线程__ 和 call-stack __执行栈__。所有的任务都会被放到执行栈等待主线程执行。
__执行栈__ 也就是在其它编程语言中所说的“调用栈”，是一种拥有 LIFO（后进先出）数据结构的栈，被用来存储代码运行时创建的所有执行上下文。当 JavaScript 引擎第一次遇到你的脚本时，它会创建一个全局的执行上下文并且压入当前执行栈。每当引擎遇到一个函数调用，它会为该函数创建一个新的执行上下文并压入栈的顶部。引擎会执行那些执行上下文位于栈顶的函数。当该函数执行结束时，执行上下文从栈中弹出，控制流程到达当前栈中的下一个上下文。
__任务队列__ Task Queue，即队列，是一种先进先出的一种数据结构。
## 同步任务和异步任务
JavaScript单线程任务被分为同步任务和异步任务。
- 同步任务会在执行栈中按照顺序等待主线程依次执行
- 异步任务会进入Event Table并注册函数。当指定的事情完成时，Event Table会将这个函数移入任务队列中。等待主线程空闲的时候（执行栈被清空），任务队列的任务按顺序被读取到栈内等待主线程的执行。
如图：
![同步任务和异步任务](/assets/img/2019/02/eventloop-1.png)

## 宏任务和微任务
除了广义的同步任务和异步任务，我们对任务有更精细的定义。在高层次上，JavaScript中有宏任务（MacroTask）和微任务（MicroTask）。
- MacroTask（宏任务）包括script全部代码、setTimeout、setInterval、I/O、UI Rendering等；
- MicroTask（微任务）包括Process.nextTick（Node独有）、Promise、Object.observe(废弃)等。

JS 引擎首先在宏任务队列中取出第一个任务`执行script`，执行完毕后取出微任务队列中的所有任务顺序执行；之后再取宏任务，如此循环，直至两个队列的任务都取完。
如图：
![宏任务和微任务](/assets/img/2019/02/eventloop-2.png)

## 总的来说
 1. 整体script作为第一个宏任务进入主线程。
 2. 同步任务被放到执行栈，异步任务会进入Event Table并注册函数，其回调函数按类别被放到宏任务队列和微任务队列中。
 3. 执行完所有同步任务后，开始读取任务队列中的结果。检查微任务队列，如果有任务则按顺序执行。
 4. 执行完所有微任务后，开始下一个宏任务。如此循环，直到两个队列（宏任务队列和微任务队列）的任务都执行完。

## 例子
``` js
setTimeout(function() {
  console.log('宏事件3');
}, 0);

new Promise((resolve) => {
  resolve(1)
  console.log('宏事件1')
}).then(function() {
  console.log('微事件1');
}).then(function() {
  console.log('微事件2');
});

console.log('宏事件2');
```
执行结果：
``` bash
宏事件1
宏事件2
微事件1
微事件2
宏事件3
```
具体过程是这样的：
 1. `执行script`任务放到宏任务队列和执行栈中，主线程执行script，`setTimeout回调函数`放到宏任务队列中，打印`宏事件1`，`Promise then1` 放到微任务队列中，打印`宏事件2`，`执行script`任务完毕，执行栈清空。
 2. 执行微任务队列的`Promise then1`，`Promise回调函数1`放到执行栈中，主线程执行`Promise回调函数1`，打印`微事件1`。回调函数返回`undefined`，此时又有then的链式调用，又放入微任务队列中，打印`微事件2`。检查微任务队列为空。
 3. 宏任务队列的`执行script`任务完毕，`setTimeout回调函数`被放到执行栈中，主线程执行，打印`setTimeout`。执行栈清空，宏任务队列清空。

## 例子2:加上async/await
``` js
console.log('script start')

async function async1() {
  await async2()
  console.log('async1 end')
}
async function async2() {
  console.log('async2 end')
}
async1()

setTimeout(function() {
  console.log('setTimeout')
}, 0)

new Promise(resolve => {
  console.log('Promise')
  resolve()
})
  .then(function() {
    console.log('promise1')
  })
  .then(function() {
    console.log('promise2')
  })

console.log('script end')
```
首先，我们需要先理解`async/await`。`async/await`实际上是`promsie`的语法糖。如下：
``` js
// async await
async function async1() {
  await async2()
  console.log('async1 end')
}
```
可以理解成
``` js
// chrome 73版本（新规范）
function async1() {
  return RESOLVE(async2).then(() => {
    console.log('async1 end')
  })
}
// chrome 73版本以下
function async1() {
  return Promise.resolve(async2).then(() => {
    console.log('async1 end')
  })
}
```
 - 在新规范中，`RESOLVE(async2)` 对于`async2`为`promise`直接返回`async2`，那么`async2`的`then`方法就会被马上调用，其回调就立即进入任务队列。
 - 而`Promise.resolve(async2)`，尽管该`promise`确定会`resolve`为`async2`，但这个过程本身是异步的，也就是现在进入任务队列的是新 `promise` 的 `resolve`过程，所以该 `promise` 的 `then` 不会被立即调用，而要等到当前任务队列执行到前述 `resolve` 过程才会被调用，然后其回调（也就是继续 `await` 之后的语句）才加入任务队列，所以时序上就晚了。

因此，在chrome 73版本中，打印的结果是：
``` bash
script start
async2 end
Promise
script end
async1 end
promise1
promise2
setTimeout
```
在chrome 73版本以下，打印的结果是：
``` bash
script start
async2 end
Promise
script end
promise1
promise2
async1 end
setTimeout
```

# Nodejs中的event loop
## 先了解一下Nodejs
### Nodejs的特点
Node.js 最大的特点就是使用 __异步式 I/O__ 与 __事件驱动__ 的架构设计。
对于高并发的解决方案，传统的架构是多线程模型，而Node.js 使用的是 __单线程__ 模型，对于所有 I/O 都使用非阻塞的异步式的请求方式，避免了频繁的线程切换。__异步式I/O__ 是这样实现的：由于大多数现代内核都是多线程的，所以它们可以处理在后台执行的多个操作。Node.js 在执行的过程中会维护一个事件队列，程序在执行时进入 __事件循环__ 等待下一个事件到来。当事件到来时，事件循环将操作交给系统内核，当一个操作完成后内核会告诉Nodejs，对应的回调会被推送到事件队列，等待程序进程进行处理。
### Nodejs的架构
![nodejs架构](/assets/img/2019/02/nodejs-1.jpg)
Node.js使用V8作为JavaScript引擎，使用高效的libev和libeio库支持事件驱动和异步式 I/O。Node.js的开发者在libev和libeio的基础上还抽象出了层libuv。对于POSIX1操作系统，libuv通过封装libev和libeio来利用 epoll 或 kqueue。在 Windows下，libuv 使用了 Windows的 IOCP机制，以在不同平台下实现同样的高性能。
Event Loop就是在libuv中实现的。
 > epoll、kqueue、IOCP都是多路复用IO接口，即支持多个同时发生的异步I/O操作的应用程序编程接口。其中epoll为Linux独占，而kqueue则在许多UNIX系统上存在，包括Mac OS X。
### Nodejs的运行机制
![nodejs运行机制](/assets/img/2019/02/nodejs.png)
Node.js的运行机制如下:
 - V8 引擎解析 JavaScript 脚本。
 - 解析后的代码，调用 Node API。
 - libuv 库负责 Node API 的执行。它将不同的任务分配给不同的线程，形成一个 Event Loop（事件循环），以异步的方式将任务的执行结果返回给 V8 引擎。
 - V8 引擎再将结果返回给用户。
 
## event loop的6个阶段
当Node.js启动时，它初始化事件循环，处理提供的输入脚本，这些脚本可能进行异步API调用、调度计时器或调用process.nextTick()，然后开始处理事件循环。
``` bash
   ┌───────────────────────┐
┌─>│        timers         │
│  └──────────┬────────────┘
│  ┌──────────┴────────────┐
│  │     pending callbacks │
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
 - timers: 执行setTimeout和setInterval中到期的callback。
 - pending callback: 上一轮循环中少数的callback会放在这一阶段执行。
 - idle, prepare: 仅在内部使用。`process.nextTick()`在这一阶段执行。
 - poll: 最重要的阶段，执行pending callback，在适当的情况下会阻塞在这个阶段。
 - check: 执行setImmediate的callback。
 - close callbacks: 执行close事件的callback，例如socket.on('close'[,fn])或者http.server.on('close, fn)。

 > setImmediate()是将事件插入到事件队列尾部，主线程和事件队列的函数执行完成之后立即执行setImmediate指定的回调函数

 event loop的每一次循环都需要依次经过上述的阶段。每个阶段都有自己的FIFO的callback队列（在timer阶段其实使用一个最小堆而不是队列来保存所有元素，比如timeout的callback是按照超时时间的顺序来调用的，并不是先进先出的队列逻辑），每当进入某个阶段，都会从所属的队列中取出callback来执行。当队列为空或者被执行callback的数量达到系统的最大数量时，进入下一阶段。这六个阶段都执行完毕称为一轮循环。

### timers
在timers阶段，会执行setTimeout和setInterval中到期的callback。执行这两者回调需要设置一个毫秒数，理论上来说，应该是时间一到就立即执行callback回调，但是由于system的调度可能会延时，达不到预期时间。如下例：
``` js
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
当进入事件循环时，它有一个空队列（`fs.readFile()`尚未完成），因此定时器将等待剩余毫秒数，当到达95ms（假设`fs.readFile()`需要95ms）时，`fs.readFile()`完成读取文件并且其完成需要10毫秒的回调被添加到轮询队列并执行。因此，原本设置100ms后执行的回调函数，会在约105ms后执行。
P.S. timers的源码[node/deps/uv/src/timer.c](https://github.com/nodejs/node/blob/master/deps/uv/src/timer.c)的uv__run_timers函数。

### pending callbacks
此阶段执行某些系统操作（例如TCP错误类型）的回调。 例如，如果TCP socket ECONNREFUSED在尝试connect时receives，则某些* nix系统希望等待报告错误。 这将在pending callbacks阶段执行。

### poll（轮询）
执行pending callback，在适当的情况下会阻塞在这个阶段。
poll阶段有两个主要功能：
 - 执行I/O（连接、数据进入/输出）回调。
 - 处理轮询队列中的事件。

当事件循环进入poll阶段并且在timers中没有可以执行定时器时，
 - 如果poll队列不为空，则事件循环将遍历其同步执行它们的callback队列，直到队列为空，或者达到system-dependent（系统相关限制）。
 - 如果poll队列为空，会检查是否有setImmediate()回调需要执行，如果有则马上进入执行check阶段以执行回调。

如果timers中有可以执行定时器且 poll 队列为空时，则会判断是否有 timer 超时，如果有的话会回到 timer 阶段执行回调。

### check
此阶段执行`setImmediate`的callback。`setImmediate()`实际上是一个特殊的计时器，它在事件循环的一个单独阶段运行。它使用一个libuv API，该API在poll阶段完成后执行callback。
 > `setImmediate()`和`setTimeout()`是相似的，但根据它们被调用的时间以不同的方式表现。
 > - `setImmediate()`设计用于在当前poll阶段完成后check阶段执行脚本 。
 > - `setTimeout()` 安排在经过最小（ms）后运行的脚本，在timers阶段执行

举个例子：
``` js
const fs = require('fs');

fs.readFile(__filename, () => {
  setTimeout(() => {
    console.log('timeout');
  }, 0);
  setImmediate(() => {
    console.log('immediate');
  });
})
```
其结果是
``` bash
immediate
timeout
```
主要原因是在I/O阶段读取文件后，事件循环会先进入poll阶段，发现有`setImmediate`需要执行，会立即进入check阶段执行`setImmediate`的回调。然后再进入timers阶段，执行`setTimeout`，打印timeout。

### close callbacks
如果套接字或句柄突然关闭(例如`socket.destroy()`)，那么'close'事件将在这个阶段发出。否则，它将通过`process.nextTick()`发出。
 > `process.nextTick()`方法将 callback 添加到next tick队列。 一旦当前事件轮询队列的任务全部完成，在next tick队列中的所有callbacks会被依次调用。即，当每个阶段完成后，如果存在 nextTick 队列，就会清空队列中的所有回调函数，并且优先于其他 microtask 执行。

# Nodejs与浏览器的Event Loop差异
 - Node 端，microtask 在事件循环的各个阶段之间执行
 - 浏览器端，microtask 在事件循环的 macrotask 执行完之后执行

![Nodejs与浏览器的Event Loop差异](/assets/img/2019/02/eventloop-3.png)

举个例子：
``` js
setTimeout(()=>{
    console.log('timer1')
    Promise.resolve().then(function() {
        console.log('promise1')
    })
}, 0)
setTimeout(()=>{
    console.log('timer2')
    Promise.resolve().then(function() {
        console.log('promise2')
    })
}, 0)
```
浏览器端运行结果：
![浏览器端运行结果：](/assets/img/2019/02/eventloop-browser.gif)
``` bash
timer1
promise1
timer2
promise2
```

node端（v10.15.1)运行结果
![node端运行结果：](/assets/img/2019/02/eventloop-node.gif)
``` bash
timer1
timer2
promise1
promise2
```
 1. 全局脚本（main()）执行，将 2 个 timer 依次放入 timer 队列，main()执行完毕，调用栈空闲，任务队列开始执行；
 2. 首先进入 timers 阶段，执行 timer1 的回调函数，打印 timer1，并将 promise1.then 回调放入 microtask 队列，同样的步骤执行 timer2，打印 timer2；
 3. 至此，timer 阶段执行结束，event loop 进入下一个阶段之前，执行 microtask 队列的所有任务，依次打印 promise1、promise2

在node新版本（v11）中，执行结果变成与浏览器一致：
``` bash
timer1
promise1
timer2
promise2
```
详情看[又被node的eventloop坑了，这次是node的锅](https://juejin.im/post/5c3e8d90f265da614274218a)

# 参考
 - [Tasks, microtasks, queues and schedules](https://jakearchibald.com/2015/tasks-microtasks-queues-and-schedules/)
 - [更快的异步函数和 Promise](https://v8.js.cn/blog/fast-async/)
 - 《Node.js开发指南》
 - [The Node.js Event Loop, Timers, and process.nextTick()](https://nodejs.org/en/docs/guides/event-loop-timers-and-nexttick/)
 - [一次弄懂Event Loop](https://mp.weixin.qq.com/s/KEl_IxMrJzI8wxbkKti5vg)
 - [不要混淆nodejs和浏览器中的event loop](https://cnodejs.org/topic/5a9108d78d6e16e56bb80882)
 - [浏览器与Node的事件循环(Event Loop)有何区别?](https://www.cnblogs.com/fundebug/p/diffrences-of-browser-and-node-in-event-loop.html)
 - [又被node的eventloop坑了，这次是node的锅](https://juejin.im/post/5c3e8d90f265da614274218a)