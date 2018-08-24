# Eventloop
对eventloopde的理解

翻译自Node官网的eventloop

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
  因为任何这些操作，可能会引入更多对操作和已经被处理的new events，这些都会被内核队列化（也就是说操作引起的新的操作或已经被处理的new events，都会在`poll phase`中排队），当poll events正在被处理，poll events能够被队列化。 结果，长时间运行的callback能够允许poll phase的运行时间超过定时器的阈值（也就是说：定时器callback被执行的延时时间=poll phase的运行时间+定时器的阈值）。详情参见timers和poll sctions。
  
**3. Phases Overview**

1）timers： 这个phase会执行`setTimeout()`跟`setInterval()`的callbacks。
2）pending callbacks: 执行异步I/O的callbacks， 会被推迟到下一次event loop去处理
3）idle，prepare: 只是内部使用
4）poll：检索新的I/O events; 执行I/O相关的callbacks(几乎所有**除了close callbacks，timers的callbacks以及setImmediate()的callback**;
5）check: 调用setImmediate()
6）close callbacks：一些close的callbacks，e.g. socket.on('close', ...)

在每次event loop之后，Nodejs会检查是否正在等待任何异步I/O或者定时器，如果没有，那么会干净地关闭。

**Phases in Detail** 
**(1) times**
    
    setTimeout(() => {
        //This is callback
    }, threshold)

一个指定了阈值的timer，在阈值到了之后，**也许**回调会被执行，但是这个时间并不是指定的阈值时间。在指定时间阈值已经到了之后（在加载的时候就会开始计时），Times callbacks将会被尽可能早地执行; 然而，操作系统的调度或者正在运行的其他回调将会使得延迟执行setTimeout的其他callbacks。
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
当event loop刚进入poll phase，这个时候queue是空的（因为fs.readFile()还没有执行完），所以event loop将等待剩余的ms数，直到定时器阈值对到达。当event loop已经等了95ms，fs.readFile()完成了文件读取（这里是分配了一个线程由libuv异步执行的），它的需要花费10ms对callback被add到poll queue，并且执行。当这个callback完成，poll queue中已经没有更多对callbacks（已经过了95ms+10ms），所以event loop将发现timer对阈值已经到了，然后event loop回到`timers phase`来执行timer对callback。在这个例子中，你将明白从timer被scheduled到它的callback被执行总共时间是105ms。

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

在poll phase已经完成之后，check phase允许我们来直接执行callbacks。如果poll phase空闲并且脚本已经用setImmediate()排队了，event loop会继续处理check phase，而不是一直在那里等待。
setImmediate()是一个特殊的timer，运行在event loop单独的phase中。在poll phase已经完成之后，它使用libuv API scheduled callbacks并执行。
一般地，执行代码对时候，event loop将最终到达poll phase，并在poll phase中等待incoming connection，request 等等。然而，如果一个callback已经被setImmediate() scheduled，并且此时poll phase是空闲的，poll phase将结束并且继续到check phase而不是一直等待poll events。

**(5) close callbacks**

如果socket或者某个处理过程突然关闭（比如：socket.destroy()），`close`事件将会在该phase被触发.`不然它将被process.nextTick()触发`。

**setImmediate() VS setTimeout()**
`setImmediate()`与`setTimeout()` 是相似的，但是会根据他们被调用的方式有不同的行为。
 * setImmediate(): 一旦poll phase完成了，就会执行check phase的 setImmediate callback.
 * setTimeout(): 在最小threshold（>= threshold）的ms数已经过去之后，将会执行timer phase的setTimeout callback.
 
setImmediate跟setTimeout这些定时器的执行顺序会在很大程度上依赖他们被调用的内容。如果他们都是在main module被调用，那么时间将跟处理性能相关（可能受到在机器上运行对其他应用的影响）。

例如，若运行下面的脚本，是在main module中，而不是在I/O的callback(I/O cycle)中,**他们的执行顺序是不确定的，因为依赖于系统的处理性能：**
```Javascipt
   // timeout_vs_immediate.js
    setTimeout(() => {
        console.log('timeout');
    }, 0);

    setImmediate(() => {
        console.log('immediate');
    }); 
```
```Javascript
    $ node timeout_vs_immediate.js
    timeout
    immediate

    $ node timeout_vs_immediate.js
    immediate
    timeout 
```
然而，**如果把上面的代码移动到I/O的callback中(I/O cycle)，immediate callback总是会先执行**
```Javascript
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
```Javascript
$ node timeout_vs_immediate.js
immediate
timeout

$ node timeout_vs_immediate.js
immediate
timeout
```
**相对于setTimeout(),使用setImmediate()的主要优势是： 在同一个I/O的callback(I/O cycle)中的任何timers，setImmediate()将总是先被执行，与存在多少计时器无关。**



**综上所述**

process.nextTick会先于下一次event loop被处理。它还有不让event loop继续的优点，比如在eventloop继续之前给用户一个警告，可能是有用的。

现在对理解是：
process.nextTick(callback)中的callback会被加到当前event loop的phase的call stack对最后面，所以会在event loop进入下一个phase之前被处理。
process.nextTick()与setImmediate()：
    * process.nextTick()直接在当前phase被触发，无论多少个嵌套都一样。
    * setImmediate()在event loop接下来对迭代过程或者tick中的check phase中触发。
*推荐开发在所有的case中使用setImmediate()，因为这样更容易推理（并且代码能与环境有更好的兼容性，比如浏览器的JS）* 

**Why use process.nextTick()？
  


关于nextTickQueue：nextTickQueue就是在使用process.nextTick(callback)的时候，将callback加入到nextTickQueue中
https://cnodejs.org/topic/4f16442ccae1f4aa2700109b

  
  
