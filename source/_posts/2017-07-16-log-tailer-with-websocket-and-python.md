---
title: 使用 WebSocket 和 Python 编写日志查看器
tags:
  - python
  - websocket
  - ops
categories: Programming
date: 2017-07-16 15:55:05
---


在生产环境运维工作中，查看线上服务器日志是一项常规工作。如果这项工作可以在浏览器中进行，而无需登录服务器执行 `tail -f` 命令，就太方便了。我们可以使用 WebSocket 技术轻松实现这一目标。在本文中，我将带各位一起使用 Python 编写一个日志查看工具。

![基于 WebSocket 的日志查看器](/images/logviewer-websocket.png)

## WebSocket 简介

WebSocket 是一个标准化协议，构建在 TCP 之上，能够在客户端和服务端之间建立一个全双工的通信渠道。这里的客户端和服务端通常是用户浏览器和 Web 服务器。在 WebSocket 诞生之前，如果我们想保持这样的一个长连接，就需要使用诸如长轮询、永久帧、Comet 等技术。而现今 WebSocket 已经得到了所有主流浏览器的支持，我们可以使用它开发出在线聊天室、游戏、实时仪表盘等软件。此外，WebSocket 可以通过 HTTP Upgrade 请求来建立连接，并使用 80 端口通信，从而降低对现有网络环境的影响，如无需穿越防火墙。

<!-- more -->

## `websockets` Python 类库

`websockets` 是第三方的 Python 类库，它能基于 Python 提供的 `asyncio` 包来实现 WebSocket 服务端以及客户端应用。我们可以使用 `pip` 来安装它，要求 Python 3.3 以上的版本。

```bash
pip install websockets
# For Python 3.3
pip install asyncio
```

下面是一段简单的 Echo 服务代码：

```python
import asyncio
import websockets

@asyncio.coroutine
def echo(websocket, path):
    message = yield from websocket.recv()
    print('recv', message)
    yield from websocket.send(message)

start_server = websockets.serve(echo, 'localhost', 8765)

asyncio.get_event_loop().run_until_complete(start_server)
asyncio.get_event_loop().run_forever()
```

可以看到，我们使用 Python 的协程来处理客户端请求。协程是 Python 3.3 引入的新概念，简单来说，它能通过单个线程来实现并发编程，主要适用于处理套接字 I/O 请求等场景。Python 3.5 开始又引入了 `async` 和 `await` 关键字，方便程序员使用协程。以下是使用新关键字对 Echo 服务进行改写：

```python
async def echo(websocket, path):
    message = await websocket.recv()
    await websocket.send(message)
```

对于客户端应用，我们直接使用浏览器内置的 `WebSocket` 类。将下面的代码直接粘贴到 Chrome 浏览器的 JavaScript 控制台中就可以运行了：

```js
let ws = new WebSocket('ws://localhost:8765')
ws.onmessage = (event) => {
  console.log(event.data)
}
ws.onopen = () => {
  ws.send('hello')
}
```

## 查看并监听日志

我们将通过以下几步来构建日志查看器：

* 首先，客户端发起一个 WebSocket 请求，并将请求的文件路径包含在 URL 中，形如 `ws://localhost:8765/tmp/build.log?tail=1`；
* 服务端接受到请求后，将文件路径解析出来，顺带解析出是否要持续监听日志的标志位；
* 服务端打开日志文件，开始不断向客户端发送日志文件内容。

完整的源代码可以在 [GitHub](https://github.com/jizhang/blog-demo/tree/master/logviewer) 中查看，以下只截取重要的部分：

```python
@asyncio.coroutine
def view_log(websocket, path):
    parse_result = urllib.parse.urlparse(path)
    file_path = os.path.abspath(parse_result.path)
    query = urllib.parse.parse_qs(parse_result.query)
    tail = query and query['tail'] and query['tail'][0] == '1'
    with open(file_path) as f:
        yield from websocket.send(f.read())
        if tail:
            while True:
                content = f.read()
                if content:
                    yield from websocket.send(content)
                else:
                    yield from asyncio.sleep(1)
        else:
            yield from websocket.close()
```

## 其它特性

* 在实际应用中发现，浏览器有时不会正确关闭 WebSocket 连接，导致服务端资源浪费，因此我们添加一个简单的心跳机制：

```python
if time.time() - last_heartbeat > HEARTBEAT_INTERVAL:
    yield from websocket.send('ping')
    pong = yield from asyncio.wait_for(websocket.recv(), 5)
    if pong != 'pong':
        raise Exception('Ping error'))
    last_heartbeat = time.time()
```

* 日志文件中有时会包含 ANSI 颜色高亮（如日志级别），我们可以使用 `ansi2html` 包来将高亮部分转换成 HTML 代码：

```python
from ansi2html import Ansi2HTMLConverter
conv = Ansi2HTMLConverter(inline=True)
yield from websocket.send(conv.convert(content, full=False))
```

* 最后，日志文件路径也需要进行权限检查，本例中是将客户端传递的路径转换成绝对路径后，简单判断了路径前缀，以作权限控制。

## 参考资料

* [WebSocket - Wikipedia](https://en.wikipedia.org/wiki/WebSocket)
* [websockets - Get Started](https://websockets.readthedocs.io/en/stable/intro.html)
* [Tasks and coroutines](https://docs.python.org/3/library/asyncio-task.html)
* [How can I tail a log file in Python?](https://stackoverflow.com/questions/12523044/how-can-i-tail-a-log-file-in-python)
