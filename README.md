# Eventloop
对eventloop对理解

翻译从官网的eventloop

##1. 什么是Event Loop
尽管JS是单线程的，但是Event Loop通过`将请求操作调度到系统内核（如果是异步操作）`，使得NodeJs能够实现`非阻塞I/O`操作。
因为大多数系统内核是多线程的，他们能够在后台同时处理多个操作。当一个异步操作完成，内核通知NodeJs将请求操作中对callback（may be？？？）添加到`poll队列`中，最终将会被执行。

##2. Event Loop解释  https://nodejs.org/en/docs/guides/event-loop-timers-and-nexttick/
当NodeJs启动，它初始化event loop，处理可能调用异步API对脚本（JS），schedule timers，或者调用process.nextTick()，然后开始处理event loop。`这就是说，process.nextTick是在每次处理event loop之前。难道执行定时器也是？？？`

下面这个简要地显示了event loop的处理顺序： **每一个环节都是event loop的 ** `phase`  

    | --->   timers
    ^          |
    |    pending callbacks
    |          |
    |     idle, prepare
    |          |              | incoming    |
    |         pool        <-- | connections |
    |          |              | data, etc   | 
    |         check
    ^          |
    |--------close callbacks
  
