---
layout: post
title: "Coroutine and asyncio"
tags: [Async, Concurrency, Library]
---

asyncio是一个用于编写并发代码的库，适合于解决I/O密集的问题。

* 目录
{:toc .toc}
---


# Term usage

- Synchronous Programming： 同步编程。

  - Single Threaded：单线程时，完成一个任务后再开始执行下一个任务。

  - Multi-Threaded：多线程时，同时处理多个任务，每一个线程的处理方式如单线程时一致。

- Asynchronous Programming：异步编程。
  - Single Threaded：单线程时，当遇到阻塞型操作（如I/O），任务被挂起并立即进行下一个任务，直到阻塞消除后再恢复。由于网络、硬盘等储存介质的读取速度与CPU相差$$10^9$$数量级以上，异步可以高效处理I/O密集型的问题。
  - Multi-Threaded：# TODO 1
  - Multi-Process: # TODO 2
- Concurrency：并发。指一个时间段内，有几个程序在**同一个**CPU上运行，但是任意时刻只有一个程序在CPU上运行。
- Parallelism：并行。指任意时刻点上，有多个程序同时运行在**多个**CPU上。受CPU的核的数量限制。
- Coroutine：协程。是一种有多个入口的**函数**，能够被暂停、恢复以及接收值。可见，Python的生成器（generator）在某程度上符合协程的要求。在协程中，`yield`往往出现在表达式的右边，例如`datum = yield x`，以实现数据的产出和接收。python在3.5版本新增了关键字`async`和`await`用来定义协程。



# a little history...

socket之间的`connect`, `recv`, `send`等通信都是典型的阻塞型I/O，下面以socket为例，简述过去解决I/O阻塞的主要方式。

## Thread

这里主要指多线程同步编程。利用多线程的确能够有效解决I/O阻塞，在处理少量而活跃的连接（比如moba游戏）时特别有用（相对的，单线程异步I/O则适合于大量而不活跃的连接，如网页浏览的场景）。但首先线程的编程难度比较高，需要程序员自己管理线程和锁。而复杂的多线程代码会让程序变得难以扩展和维护。更重要的是，线程启动时的开销比较大，因此才有了著名的C10K problem[^c10k]。对于有着千万级连接的大规模应用而言，内存相对线程的开销将显得捉襟见肘。



## Async

Python的`socket`模块可以将socket设置成非阻塞：

```python
sock = socket.socket()
sock.setblocking(False)  			# 设置为非阻塞
try:
    sock.connect(('xkcd.com', 80))  # 设置为非阻塞后，connect被调用后会立即返回
except BlockingIOError:				# 并抛出BlockingIOError
    pass

request = 'GET {} HTTP/1.0\r\nHost: xkcd.com\r\n\r\n'.format(url)
encoded = request.encode('ascii')

while True:
    try:
        sock.send(encoded)
        break  # Done.
    except OSError as e:
        pass

print('sent')
```

在上面的例子中，为了确认连接建立，用了一个While循环做轮询。这种做法不但消耗大量CPU资源，也无法处理多个socket的情况，因此无太大意义。

传统的办法是利用`select`的函数来解决：

```python
from selectors import DefaultSelector, EVENT_WRITE

selector = DefaultSelector()			# 根据系统自动选择最适合的select函数

sock = socket.socket()
sock.setblocking(False)
try:
    sock.connect(('xkcd.com', 80))
except BlockingIOError:
    pass

def connected():						# 回调函数
    selector.unregister(sock.fileno())
    print('connected!')

selector.register(sock.fileno(), EVENT_WRITE, connected)
```

`select`会等待事件的发生，然后执行里面的回调函数。



## Callbacks

在实际应用中，回调函数代码的复杂性非常高，发生异常时也难以分析，影响代码的可读性和可维护性。严重时，江湖人称回调地狱（callback hell）:

```python
def stage1(response1):
    request2 = step1(response1)
    api_call2(request2, stage2)

def stage2(response2):
    request3 = step2(response2)
    api_call3(request3, stage3)

def stage3(response3):
    step3(response3)

api_call1(request1, stage1)
```



## Coroutines

协程即拥有回调的高效，还有简洁的代码：

```python
@asyncio.coroutine
def fetch(self, url):
    response = yield from self.session.get(url)
    body = yield from response.read()
```

协程在Python中是基于生成器、Future类、事件循环来实现的。

### yield & yield from

#### yield

在协程中，`yield`通常出现在表达式的右边（例如`datum = yield ...`，如果`yield`后面没有表达式，那么生成器产出`None`）。这时候，调用方可通过`.send(datum)`的方法把值推送给协程，但只能在协程被激活并且处于暂停状态时才能调用`send`：

```python
>>> def cor():
...     print("coroutin started")
...     x = yield		# 产出值为None
...     print("coroutin received:", x)
...     y = yield 'y'
...     print("coroutin received:", y)
... 
>>> cor = cor()
>>> cor.send(None)		# 预激（prime）协程，执行至第一个yield语句处暂停
coroutin started
>>> cor.send('x')		# 'x'被发送成为暂停位置yield表达式的值。协程恢复，运行至下一个yield
coroutin received: x
'y'						# 继续运行至下一个yield，返回产出的值’y'
>>> cor.send('z')		# ‘z'被赋值给变量z
coroutin received: z
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
StopIteration			# 协程终止，抛出StopIteration
```

生成器有两个方法可以显式地抛出异常：

- `generator.throw`，使生成器在暂停的`yield`表达式处抛出指定的异常。如果异常被处理，代码会向前执行到下一个`yield`表达式。TODO 3：如果异常没有被处理，异常会向上冒泡，传到调用方的上下文中。
- `generator.close`，使生成器在暂停的`yield`表达式处抛出`GeneratorExit`异常。如果生成器没有处理这个异常，或者抛出了`StopIteration`异常，调用方不会报错。

#### yield from

`yield from`是python3.3版本新增的语法。它能够在调用方和子生成器之间建立双向的连接。也就是说，调用方和子生成器可以直接发送和产出值，还可以直接传入异常，而不用在位于中间的协程中添加大量处理异常的代码。

> 委派生成器：包含`yield from <iterable>`表达式的生成器函数。
>
> 子生成器：从`yield from`表达式中`<iterable>`获取的生成器。可以是实现了`__next__`, `send`, `close`, `throw`方法的生成器，也可以是只实现了`__next__`的简单的迭代器。
>
> 调用方：委派生成器的调用者。

值得注意的是，如果子生成器不终止，委派生成器就会在`d = yield from`表达式处永远暂停，意味着d得不到赋值。以下是计算给定数据的平均值的例子，只列出相关代码：

```python
# 子生成器
def averageer():
    total = 0.0
    count = 0
    average = None
    while True:
        term = yield
        if term is None:			# 0.子生成器的终止条件，必须含有
            break				
        total += term
        count += 1
        average = total / count
    return Result(count, average)	# 4.一旦运行，触发StopIteration，委派生成器将恢复运行。

# 委派生成器
def grouper(results, key):
    while True:
        results[key] = yield from averageer()  # 5.results[key]获得子生成器的返回值

# 调用方
def main(data):
    results = {}
    for key, values in data.items():
        group = grouper(results, key)
        next(group)					# 1.预激，在yield from处暂停
        for value in values:
            group.send(value)		# 2.值将会直接传给子生成器
        group.send(None)			# 3.导致averager()终止，委派生成器恢复运行
    report(results)

# ...
```

另外，`yield from`处理异常的机制较为复杂，详见：[PEP 380](https://www.python.org/dev/peps/pep-0380/)。

### Future, Task and Event loop

`future`、`task`和`Event loop`都是异步编程中的核心理念。

#### future

也被称为未来对象，是一个task的返回容器。它的实例表示可能已经完成或者尚未完成的延迟计算。官方文档说它代表的是一个终将由异步操作产生的结果（A Future represents an eventual result of an asynchronous operation. Not thread-safe.）。这与Twisted引擎中的Deferred类，Tornado框架中的Future类，以及多个JavaScript库中的Promise对象类似。

`future`封装待完成的操作，可以放入队列，完成的状态可以查询，得到结果（或抛出异常）后可以获取结果（或异常）。

Python有两个Future的类：`concurrent.futures.Future`和`asyncio.Future`。它们的接口基本一致，但实现方法不一样，往后版本可能会统一起来。本文只关注`asyncio.Future`。

#### task

`Task`是一个被包裹在`Future`的协程（A coroutine wapped in a Future），实际上是`Future`的子类。它充当了Future与协程之间的桥梁，驱动协程前进：如果被包裹的协程`yield from`一个future，task会暂停直到该future完成。

`task`需要放在`Event loop`中调度和运行，每当`task`暂停，`Event loop`就会启动另一个`task`。



下面是它们的简化版本，并非`asyncio`包中的真实代码：

```python
class Future:
    def __init__(self):
        self.result = None			# 初始化的future处于pending状态
        self._callbacks = []

    def add_done_callback(self, fn):
        '''添加一个回调函数，待future完成后调用'''
        self._callbacks.append(fn)

    def set_result(self, result):
        '''使future完成，并设置它的结果'''
        self.result = result		# 设置结果
        for fn in self._callbacks:	# 调用callbacks
            fn(self)				# 回调函数接受的参数是future本身

class Task:
    def __init__(self, coro):
        self.coro = coro
        f = Future()			# 创建一个初始化的future
        f.set_result(None)
        self.step(f)

    def step(self, future):
        try:					# 驱动协程前进，返回的future被next_future捕获
            next_future = self.coro.send(future.result)	
        except StopIteration:	# 协程运行到终点，抛出异常，返回
            return

        next_future.add_done_callback(self.step)
            
class Fetcher:
    def fetch(self):						# 协程1
        sock = socket.socket()
        sock.setblocking(False)
        try:
            sock.connect(('xkcd.com', 80))	# 阻塞I/O 1
        except BlockingIOError:
            pass

        f = Future()						# 创建一个future，用来接收连接后返回的结果

        def on_connected():
            f.set_result(None)				# 连接后，future将结果设置为None，

        selector.register(sock.fileno(),
                          EVENT_WRITE,
                          on_connected)		# 连接建立后，调用step(f),驱动协程1前进
        yield f 							# 协程被预激后暂停于此处
        selector.unregister(sock.fileno())	# 连接完成，注销sock，
        print('connected!')
        
        sock.send(request.encode('ascii'))			# 阻塞I/O 2
        self.response = yield from read_all(sock)
        
        def read(sock):								# 协程2
        	f = Future()							# 新建future以接收读取的结果

            def on_readable():
                f.set_result(sock.recv(4096))		# 阻塞I/O 3

            selector.register(sock.fileno(), EVENT_READ, on_readable) # step继续驱动协程前进
            chunk = yield f  						# chunk接收sock.recv(4096)的数据
            selector.unregister(sock.fileno())
            return chunk							# 协程结束，返回chunk
    
        def read_all(sock):					# 协程3
        response = []
        # Read whole response.
        chunk = yield from read(sock)		# 读取一次，返回值被chunk捕获
        while chunk:						# 如果数据量大，需要多次读取
            response.append(chunk)
            chunk = yield from read(sock)

        return b''.join(response)
        
```

*备注：通常情况下都应避免自己创建future，而是交给并发框架去实例化*

#### Event loop

Event loop，事件循环，是一个程序结构，用于等待和发送消息和事件。可以简单理解为在程序中设置两个线程，一个负责程序运行本身，即“主线程”；另一个负责主线程与其他进程（主要是各种I/O操作）的通信，即“Event loop线程”。每当遇到I/O的时候，主线程就让Event Loop线程去通知相应的I/O程序，然后接着往后运行。等到I/O程序完成操作，Event Loop线程再把结果返回主线程。主线程就调用事先设定的回调函数，完成整个任务。

在Python中，协程不能像函数那样被直接调用，而必须使用事件循环显式排定协程的执行时间（或者在其他排定了执行时间的协程中使用`yield from`/`await`表达式来激活）。

在python3.7以前，事件循环通过`loop = asyncio.get_event_loop()`获取，调用`loop.run_until_complete(future)`运行直到future完成。参数也可以接受协程对象，它会自动把协程包装在Task对象中。（或者通过`asyncio.ensure_future(coro_or_future)`，再调用`loop.run_forever()`。但是停止需要在协程对应的位置调用`loop.stop()`。）

**3.7以后，可以直接调用`asyncio.run(coro)`运行协程。**



# asyncio

asyncio省去了上述的许多麻烦：TODO

```python
async def fetch(url):
    await asyncio.sleep(1)			# 模拟connect

    async def sub_coro1():			# 子协程1
        res = await sub_coro2()
        return res

    async def sub_coro2():			# 子协程2
        await asyncio.sleep(1)
        return url + 'result'

    res = await sub_coro1()
    print(res)

tasks = [fetch(i) for i in '1234']
asyncio.run(asyncio.wait(tasks))	# wait()将由future或协程构成的可迭代对象包装成一个Task对象
```



## APIs

## High-level APIs

#### asyncio.create_task(coro)

将协程包裹到一个Task中并返回，会被自动提交到事件循环中以调度其并发的执行时间：

```python
async def say_after(delay, what):
    await asyncio.sleep(delay)
    print(what)

async def main():
    print(f"started at {time.strftime('%X')}")
    
    # await say_after(1, 'hello')
    # await say_after(2, 'world')
    
    task1 = asyncio.create_task(
        say_after(1, 'hello'))

    task2 = asyncio.create_task(
        say_after(2, 'world'))
    
    await task1		# 两个task都被提交到事件循环
    await task2		# 执行时间约为2s，比上面被注释掉的方式快1s

    print(f"finished at {time.strftime('%X')}")

asyncio.run(main())
```

另外，如有需要，task1和task2可以调用`result()`获得`say_after`的返回值。

在3.7以前的版本，`creat_task(coro)`可以用`ensure_future(coro())`代替，这是一个low-level的版本。



#### asyncio.gather(*aws, loop=None, return_exceptions=False)

aws是awaitable对象。如果aws任何一个都是协程，就会被自动打包成一个Task。这些aws将被并发运行。全部完成后，返回一个包含所有返回值（按顺序排列）的列表。



#### asyncio.wait(*aws*, ***, *loop=None*, *timeout=None*, *return_when=ALL_COMPLETED*)

类似`gather`，`wait`打包运行aws里的awaitable对象。keyword `return_when`有3个参数可选，默认的ALL_COMPLETED, 另外时FIRST_COMPLETED和FIRST_EXCEPTION。顾名思义，分别会在以下情形返回：全部future完成或取消、第一个future完成或取消和第一个future完成并抛出异常（如无异常则与ALL_COMPLETED一样）。



## Low-level APIs



[^c10k]: <http://www.kegel.com/c10k.html>

