---
title: Log Tailer with WebSocket and Python
tags:
  - python
  - websocket
  - ops
categories: Programming
date: 2017-07-15 19:21:03
---


Tailing a log file is a common task when we deploy or maintain some software in production. Instead of logging into the server and `tail -f`, it would be nice if we can tail a log file in the browser. With WebSocket, this can be done easily. In this article, I'll walk you through a simple **logviewer** ([source](https://github.com/jizhang/blog-demo/tree/master/logviewer)) utility that is written in Python.

![Logviewer with WebSocket](/images/logviewer-websocket.png)

## WebSocket Intro

WebSocket is standard protocol over TCP, that provides full-duplex communication between client and server side, usually a browser and a web server. Before WebSocket, when we want to keep an alive browser-server connection, we choose from long polling, forever frame or Comet techniques. Now that WebSocket is widely supported by major browsers, we can use it to implement web chatroom, games, realtime dashboard, etc. Besides, WebSocket connection can be established by an HTTP upgrade request, and communicate over 80 port, so as to bring minimum impact on existing network facility.

<!-- more -->

## Python's `websockets` Package

`websockets` is a Python package that utilize Python's `asyncio` to develop WebSocket servers and clients. The package can be installed via `pip`, and it requires Python 3.3+.

```bash
pip install websockets
# For Python 3.3
pip install asyncio
```

Following is a simple Echo server:

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

Here we use Python's coroutines to handle client requests. Coroutine enables single-threaded application to run concurrent codes, such as handling socket I/O. Note that Python 3.5 introduced two new keywords for coroutine, `async` and `await`, so the Echo server can be rewritten as:

```python
async def echo(websocket, path):
    message = await websocket.recv()
    await websocket.send(message)
```

For client side, we use the built-in `WebSocket` class. You can simply paste the following code into Chrome's JavaScript console:

```js
let ws = new WebSocket('ws://localhost:8765')
ws.onmessage = (event) => {
  console.log(event.data)
}
ws.onopen = () => {
  ws.send('hello')
}
```

## Tail a Log File

We'll take the following steps to implement a log viewer:

* Client opens a WebSocket connection, and puts the file path in the url, like `ws://localhost:8765/tmp/build.log?tail=1`;
* Server parses the file path, along with a flag that indicates whether this is a view once or tail request;
* Open file and start sending contents within a for loop.

Full code can be found on [GitHub](https://github.com/jizhang/blog-demo/tree/master/logviewer), so here I'll select some important parts:

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

## Miscellaneous

* Sometimes the client browser will not close the connection properly, so it's necessary to add some heartbeat mechanism. For instance:

```python
if time.time() - last_heartbeat > HEARTBEAT_INTERVAL:
    yield from websocket.send('ping')
    pong = yield from asyncio.wait_for(websocket.recv(), 5)
    if pong != 'pong':
        raise Exception('Ping error'))
    last_heartbeat = time.time()
```

* Log files may contain ANSI color codes (e.g. logging level). We can use `ansi2html` package to convert them into HTML:

```python
from ansi2html import Ansi2HTMLConverter
conv = Ansi2HTMLConverter(inline=True)
yield from websocket.send(conv.convert(content, full=False))
```

* It's also necessary to do some permission checks on the file path. For example, convert to absolute path and do a simple prefix check.

## References

* [WebSocket - Wikipedia](https://en.wikipedia.org/wiki/WebSocket)
* [websockets - Get Started](https://websockets.readthedocs.io/en/stable/intro.html)
* [Tasks and coroutines](https://docs.python.org/3/library/asyncio-task.html)
* [How can I tail a log file in Python?](https://stackoverflow.com/questions/12523044/how-can-i-tail-a-log-file-in-python)
