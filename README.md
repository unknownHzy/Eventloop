# Eventloop
对eventloop对理解

翻译从官网的eventloop

**1. 什么是Event Loop**
尽管JS是单线程的，但是Event Loop通过`将请求操作调度到系统内核（如果是异步操作）`，使得NodeJs能够实现`非阻塞I/O`操作。
因为大多数系统内核是多线程的，他们能够在后台同时处理多个操作。当一个异步操作完成，内核通知NodeJs将请求操作中对callback（may be？？？）添加到`poll队列`中，最终将会被执行。

**2. Event Loop解释**  https://nodejs.org/en/docs/guides/event-loop-timers-and-nexttick/
当NodeJs启动，它初始化event loop，处理可能调用异步API对脚本（JS），schedule timers，或者调用process.nextTick()，然后开始处理event loop。`这就是说，process.nextTick是在每次处理event loop之前。难道执行定时器也是？？？`

下面这个简要地显示了event loop的处理顺序：**每一个环节都是event loop的**`phase`  

    | --->   timers
    ^          |
    |    pending callbacks
    |          |
    |     idle, prepare
    |          |              | incoming    |
    |         poll        <-- | connections |
    |          |              | data, etc   | 
    |         check
    ^          |
    |--------close callbacks
  
  每一个`phase`是一个callbacks的FIFO队列，当event loop进入一个指定phase，一般的，将首先执行针对该phase的任何操作，然后执行phase队列中对callbacks，直到队列全部执行完或者最大数量对callbacks被执行。当这个phase队列已经被耗尽或者callbacks上限已到，event loop将移动到下一个phase。
  因为任何这些操作，可能会引入更多对操作和已经被处理的new events，这些都会被内核队列化（也就是说操作引起的新的操作或已经被处理的new events，都会在`poll phase`中排队），当poll events正在被处理，poll events能够被队列化。 结果，长时间运行的callback能够允许poll phase对运行时间超过定时器对阈值。详情参见timers和poll sctions。
  
**3. Phases Overview**
1）timers： 这个phase会执行`setTimeout()`跟`setInterval()`对callbacks
2）pending callbacks: 执行异步I/O的callbacks， 会被推迟到下一次event loop
3）idle，prepare: 只是内部使用
4）poll：取回新的I/O events; 执行I/O相关的callbacks(几乎所有**除了close callbacks，timers的callbacks以及setImmediate()的callback**;
5）check: 调用setImmediate()
6）close callbacks：一些close的callbacks，e.g. socket.on('close', ...)

在每次event loop之后， Nodejs会检查是否正在等待任何异步I/O或者定时器，如果没有，那么会干净地关闭。

**Phases in Detail** 
**(1) times**
    
    setTimeout(() => {
        //This is callback
    }, threshold)

一个指定了时间阈值的timer，在时间阈值到了之后，**也许**回调会被执行，但是这个时间并不是指定的阈值时间。在指定时间阈值已经到了之后，Times callbacks将会被尽可能早地执行; 然而，操作系统的调度或者正在运行的其他回调将会使得延迟执行setTimeout的其他callbacks。
**setTimeout中callback的执行时间 = threshold + 操作系统调度时间或者其他回调的执行时间（或者两者之和）**
    
```Javascript
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
当event loop刚进入poll phase，这个时候queue是空的（因为fs.readFile()还没有执行完），所以event loop将等待剩余对ms数，直到定时器阈值对到达。当event loop已经等了95ms，fs.readFile()完成了文件读取（这里是分配了一个线程由libuv异步执行的），它的需要花费10ms对callback被add到poll queue，并且执行。当这个callback完成，poll queue中已经没有更多对callbacks（已经过了95ms+10ms），所以event loop将发现timer对阈值已经到了，然后event loop回到`timers phase`来执行timer对callback。在这个例子中，你将明白从timer被scheduled到它的callback被执行总共时间是105ms。

*Node: 为了防止poll phase阻塞住event loop，在libuv（实现了Nodejs的event loop和平台所有的异步行为的C库）为更多的events停止轮寻的之前，libuv有一个依赖系统的轮寻上限值，到达这个值之后，不管后面是否继续轮寻，就直接停掉了。*
    
**(2) pending callbacks**
这个phase为系统操作执行callbacks，例如TCP errors类型。比如当尝试连接的时候，如果一个TCP socket接收到`ECONNREFUSED`，一些*nix系统想要等到错误被报出来。这将在pending callbacks队列化被执行。

**(3) poll**
poll phase有两个主要功能：
    1. 计算poll phase将会阻塞event loop多久，并且为I/O轮寻，然后
    2. 处理poll队列中对events

当event loop进入poll phase的时候（假设此时没有timers被scheduled），下面其中之一的情形将发生：
    * 如果poll队列`不为空`，event loop将**同步地执行**整个队列中callbacks，直到整个队列中的callbacks已经被执行完，或者系统上限已经到达。
    * 如果poll队列`为空`，以下情形之一将发生：
        ** 如果脚本中已经被`setImmediate()` scheduled了，event loop将会结束poll phase，往下走到check phase执行这些scheduled脚本
        ** 如果脚本中还没有被`setImmediate()` scheduled，event loop将会等待callbacks被增加到队列中，然后直接执行他们。
        
 一旦poll队列空了，event loop将会检查timers（假设之前有timers被scheduled），看哪个timer的时间阈值已经到了。如果一个或者更多的timers时间阈值已经到了，event loop将会回撤到timers phase来执行这些timers的callbacks。
 
**(4) check**
在poll phase已经完成之后，会直接地执行check phase的callbacks。如果poll phase变得空闲，




**综上所述**
process.nextTick会先于下一次event loop被处理。即会排在event queue的最后。
  
  