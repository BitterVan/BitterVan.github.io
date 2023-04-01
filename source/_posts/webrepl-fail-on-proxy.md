---
title: WebSocket Cannot Be Established Due to Proxy
date: 2022-03-31 10:20:15
tags: Embedded System
---

I've been working on building a visual studio code extension for a better-developing experience on ESP32, MicroPython. But without any basic knowledge of front-end development, this process is quite tough for me.

To establish a wireless connection, the first thing that came to my mind is that I should check the official tools. So I checked the source code of project [webrepl](https://github.com/micropython/webrepl), a WebREPL terminal client, which also has a coarse front-end written in Javascript.

<!-- more -->

## Problem

It seemed to be simple to connect to the webrepl server running on the board, as the code in [webrepl.html](https://github.com/micropython/webrepl/blob/master/webrepl.html) showed. But when I was using the npm package `ws` or `websocket`  in Typescript to connect to it, the connection was simply cannot be established. The error in the serial connection illustrated that

```
Traceback (most recent call last):
  File "webrepl.py", line 43, in accept_conn
  File "websocket_helper.py", line 39, in server_handshakej
OSError: Not a websocket request

WebREPL connection from: ('192.168.137.1', 2935)

```

<!-- more -->
At first, I thought it was just like ordinary problems, parameters, policies, etc. So I applied WireShark to check the difference between the packets sent by webrepl and the Typescript extension written by myself.

Using the `ws` package to connect:

![Packet Using `ws`](ws_pack.png)

Using Javascript `WebSocket` to connect to the board:

![Packet Using JavaScript](js_pack.png)

Having no idea which options to alter, I referred to the [MicroPython](https://github.com/micropython/micropython) project to find out. But the code shows that’s nothing to do with the options.

```python
webkey = None

while 1:
    l = clr.readline()
    if not l:
        raise OSError("EOF in headers")
    if l == b"\r\n":
        break
    #    sys.stdout.write(l)
    h, v = [x.strip() for x in l.split(b":", 1)]
    if DEBUG:
        print((h, v))
    if h == b"Sec-WebSocket-Key":
        webkey = v

if not webkey:
     raise OSError("Not a websocket request")
```

But how can `webkey == None` happen? The two packets both have `Sec-WebSocket-Key` in their fields.

## Solution

Fortunately, one of my teammates found that the client was assigned with `Go-client`, and it was our proxy server [Clash For Windows](https://github.com/Fndroid/clash_for_windows_pkg) that modified our packet. And turning off the proxy worked.