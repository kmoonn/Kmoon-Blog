+++
title = "SSE：服务端实时推送技术"
date = "2025-11-06T17:15:20+08:00"
tags = ["SSE", "Go"]
description = "Server-Sent Events 的工作原理、与 WebSocket 的对比，以及如何用 Go 实现一个支持断线重连的生产级 SSE 服务。"
+++

如果你用过 ChatGPT 或 Claude，一定见过这种效果：AI 回复不是一次性出现的，而是一个字一个字地"蹦"出来。你问一句话，它开始打字，像真人在聊天框另一边敲键盘。

这背后用的技术就是 SSE（Server-Sent Events）。一个诞生了十几年的老协议，在 2025 年因为大模型流式响应的普及，突然又回到了聚光灯下。

## SSE 是什么

SSE 做的事情很简单：**服务器向浏览器单向推送数据，通过一个长连接持续发送事件流**。

从协议角度看，它比 WebSocket 简单得多。客户端发一个普通的 HTTP GET 请求，带上 `Accept: text/event-stream` 头。服务器返回 `Content-Type: text/event-stream`，然后连接就不关了——服务器一直往里面写数据，直到某一方主动断开。

```http
GET /events HTTP/1.1
Host: example.com
Accept: text/event-stream

HTTP/1.1 200 OK
Content-Type: text/event-stream
Cache-Control: no-cache
Connection: keep-alive

data: {"message": "第一条消息"}

data: {"message": "第二条消息"}

```

每条消息以 `data:` 开头，用两个换行符 `\n\n` 分隔。格式简单到你可以用 `curl` 直接连上看：

```bash
curl -N http://localhost:8080/events
```

## SSE vs WebSocket：别一上来就上 WebSocket

做后端的人遇到"实时推送"需求，第一反应往往是 WebSocket。我以前也这样。但大多数场景其实不需要双向通信——你只是想从服务器往客户端推数据而已。

打个比方：SSE 像收音机广播，服务器是电台，客户端是听众，电台只管播、听众只管听。WebSocket 像打电话，双方都能说、都能听，但也需要双方都维护连接状态。

| 维度 | SSE | WebSocket |
|------|-----|-----------|
| 通信方向 | 服务器 → 客户端 | 双向 |
| 协议 | HTTP | ws:// / wss:// |
| 每消息开销 | ~5 bytes | ~2 bytes |
| 自动重连 | 浏览器内置 | 需要自己实现 |
| 二进制数据 | 不支持（仅文本） | 原生支持 |
| HTTP/2 多路复用 | 支持 | 不支持（独立协议） |
| 客户端 API | `EventSource` | `WebSocket` |

WebSocket 的单条消息开销更低（~2 bytes vs ~5 bytes）、支持二进制帧，这些是它的优势。但 SSE 有两个你写代码时能省不少事的特性：

**一是自动重连**。`EventSource` API 在连接断开后会自动重试，不需要写任何重连逻辑。你还可以通过 `Last-Event-ID` 这个 HTTP 头告诉服务器"上次我收到第几条了"，服务端据此推送断开期间漏掉的事件。而 WebSocket 的重连你得自己写，漏掉的消息你得自己补。

**二是走标准 HTTP**。这意味着它可以透明穿过所有 HTTP 代理、负载均衡器、CDN。WebSocket 需要中间件显式支持 `Upgrade` 机制，在一些企业网络环境里会被防火墙拦截。说白了，SSE 的部署难度基本为零——你的 HTTP 服务能跑，SSE 就能跑。

那什么时候该用 WebSocket？需要双向通信的时候——比如协作编辑、在线游戏的实时操作、聊天应用的消息收发。但如果你的场景是"服务器有更新了通知客户端"——AI 流式输出、系统监控、消息通知、日志推送——SSE 足以胜任，而且省事太多。

## 协议细节：event、data、id、retry

SSE 消息格式很简单，但有四个关键字段值得单独讲清楚：

**`data:`**——消息体，最核心的字段。你可以写多行 `data:`，浏览器会把它们拼接成一个完整消息（用换行符连接）。这也是 SSE 的"多行数据"机制——每多一行 `data:`，最终消息里就多一个换行。

```text
data: 第一行
data: 第二行
data: 第三行

```

上面这段发给客户端时，`onmessage` 拿到的就是 `第一行\n第二行\n第三行`。

**`event:`**——事件类型。如果不写 `event:`，浏览器走默认的 `onmessage` 回调。写了的话，就需要用 `addEventListener('事件名', ...)` 来监听。这让你可以在同一条 SSE 连接上区分不同类型的推送——比如 `event: notification` 和 `event: log` 走不同的处理逻辑。

```text
event: notification
data: {"title": "告警", "body": "CPU 超过 90%"}

event: log
data: {"level": "INFO", "message": "任务完成"}

```

**`id:`**——事件编号，断线重连的基石。浏览器收到 `id:` 后会在内部记录这个值。连接断开再重连时，自动带上 `Last-Event-ID: <上次收到的id>` 请求头。服务器读到这个头就知道从哪开始补推，不需要客户端额外处理。

```text
id: 42
data: {"message": "第42条消息"}

```

**`retry:`**——重连间隔，单位毫秒。服务器通过这个字段告诉浏览器"断了之后等多久再试"。默认是 3 秒左右（各浏览器实现略有差异），你可以根据自己的场景调整——数据更新频繁的可以设短一点，资源敏感的设长一点。

```text
retry: 5000

```

另外还有一个非正式的约定：以 `:` 开头的行是注释，浏览器会忽略。这被广泛用于**心跳保活**——每 30 秒发一个 `: heartbeat`，中间代理和负载均衡器看到有数据在流动就不会关连接。

## Go 实现：一个生产可用的 Broker 模式

说到写代码，SSE 在服务端的实现核心就是一个**事件分发器**（Event Broker）。服务器维护一个客户端列表，有新消息时遍历列表推送给每个客户端。

下面是一个 Go 语言的实现，涵盖了生产环境的关键关注点——非阻塞发送、心跳保活、断线重连、优雅关闭：

```go
package main

import (
    "fmt"
    "log"
    "net/http"
    "sync"
    "time"
)

// EventBroker 负责管理所有 SSE 客户端连接
type EventBroker struct {
    clients    map[chan []byte]struct{}
    register   chan chan []byte
    unregister chan chan []byte
    broadcast  chan []byte
    mu         sync.RWMutex
}

func NewEventBroker() *EventBroker {
    return &EventBroker{
        clients:    make(map[chan []byte]struct{}),
        register:   make(chan chan []byte),
        unregister: make(chan chan []byte),
        broadcast:  make(chan []byte),
    }
}

// Run 在单个 goroutine 中串行处理所有客户端操作，避免锁竞争
func (b *EventBroker) Run() {
    for {
        select {
        case client := <-b.register:
            b.clients[client] = struct{}{}

        case client := <-b.unregister:
            if _, ok := b.clients[client]; ok {
                delete(b.clients, client)
                close(client)
            }

        case msg := <-b.broadcast:
            for client := range b.clients {
                // 非阻塞发送：慢客户端不会拖死其他客户端
                select {
                case client <- msg:
                default:
                    // 客户端消费太慢，跳过这条消息
                }
            }
        }
    }
}

// sseHandler 处理单个 SSE 客户端连接
func (b *EventBroker) sseHandler(w http.ResponseWriter, r *http.Request) {
    // 设置 SSE 必需的响应头
    w.Header().Set("Content-Type", "text/event-stream")
    w.Header().Set("Cache-Control", "no-cache")
    w.Header().Set("Connection", "keep-alive")
    w.Header().Set("X-Accel-Buffering", "no") // 防止 Nginx 缓冲

    flusher, ok := w.(http.Flusher)
    if !ok {
        http.Error(w, "不支持 Streaming", http.StatusInternalServerError)
        return
    }

    // 读取客户端最后收到的事件 ID（断线重连场景）
    lastEventID := r.Header.Get("Last-Event-ID")

    client := make(chan []byte, 256) // 带缓冲，减少阻塞
    b.register <- client
    defer func() {
        b.unregister <- client
    }()

    // 心跳 ticker：每 30 秒发一次注释行，防止代理断开空闲连接
    heartbeat := time.NewTicker(30 * time.Second)
    defer heartbeat.Stop()

    // 通知客户端连接已建立
    fmt.Fprintf(w, "retry: 3000\n\n")
    flusher.Flush()

    // 如果有断线重连的需求，补推丢失的事件
    if lastEventID != "" {
        fmt.Fprintf(w, "id: %s\ndata: 重连成功，从 ID %s 继续\n\n", lastEventID, lastEventID)
        flusher.Flush()
    }

    ctx := r.Context()

    for {
        select {
        case <-ctx.Done():
            // 客户端断开连接
            return

        case <-heartbeat.C:
            // 心跳：SSE 注释行，浏览器忽略但能保活
            fmt.Fprintf(w, ": heartbeat\n\n")
            flusher.Flush()

        case msg, ok := <-client:
            if !ok {
                return
            }
            fmt.Fprintf(w, "data: %s\n\n", msg)
            flusher.Flush()
        }
    }
}

func main() {
    broker := NewEventBroker()
    go broker.Run()

    http.HandleFunc("/events", broker.sseHandler)

    // 模拟推送：每 2 秒广播一条消息
    go func() {
        ticker := time.NewTicker(2 * time.Second)
        for range ticker.C {
            msg := fmt.Sprintf("服务器时间: %s", time.Now().Format("15:04:05"))
            broker.broadcast <- []byte(msg)
        }
    }()

    log.Println("SSE 服务启动在 :8080")
    log.Fatal(http.ListenAndServe(":8080", nil))
}
```

这段代码里值得单独说的几个点：

**非阻塞发送**（`select default` 分支）。这是 broker 模式里最容易踩的坑——如果某个客户端网络卡了就阻塞了 `broadcast` goroutine，所有客户端都收不到消息。用 `default` 跳过慢客户端，它自会被心跳检测到然后清理掉。

**`X-Accel-Buffering: no`** 这个头。Nginx 默认会缓冲 upstream 的响应，不设这个头的话，你的 SSE 事件可能被 Nginx 攒够 4KB 才一次性发出去，实时性全没了。生产环境用 Nginx 反代的，记得加上。

**`retry:` 在连接时发送**。客户端拿到 `retry: 3000` 后，断了会在 3 秒后重连。这比依赖浏览器默认值更可控——默认值各浏览器不一样，Safari 和 Chrome 就不一致。

客户端代码反而简单得多：

```javascript
const es = new EventSource('/events')

es.onmessage = (event) => {
  const data = JSON.parse(event.data)
  console.log('收到消息:', data)
}

es.addEventListener('notification', (event) => {
  // 处理特定类型的事件
  console.log('通知:', JSON.parse(event.data))
})

es.onerror = (event) => {
  // EventSource 会自动重连，这里只做日志
  console.warn('SSE 连接异常，正在自动重连...')
}

// 需要断开时
// es.close()
```

`EventSource` API 的自动重连是内置的——你不需要写任何 reconnect 逻辑，开了就完事。`onerror` 回调只是让你知道重连正在发生，不需要手动干预。

## HTTP/2 下 SSE 的质变

SSE 在 HTTP/1.1 时代有一个很麻烦的限制：**浏览器对同一个域名的并发连接数上限是 6 个**（这是 Chrome 和 Firefox 的硬编码限制，标记为"Won't Fix"至今）。如果你开了 6 个 SSE 连接，那这个域名的其他所有请求——图片、CSS、XHR——全部排队等，页面直接卡死。

这意味着在 HTTP/1.1 下，你几乎只能维护一个 SSE 连接，并且得把所有推送频道塞进这一个连接里。

HTTP/2 的多路复用彻底解决了这个问题。多个 stream 共享一个 TCP 连接，默认支持 100 个并发 stream，而 SSE 每个连接只占一个 stream。你可以在一个页面里同时连多个 SSE 端点，同时还正常加载其他资源，互不干扰。

再加上 HTTP/2 的服务端推送（Server Push）虽然没大规模落地，但 SSE 天然就是"应用层的 Server Push"——它不需要 HTTP/2 Push 那些复杂的缓存策略和优先级控制，直接走应用协议自己管理。

这也是为什么 2025 年 SSE 重新火起来的原因之一——HTTP/2 的普及把 SSE 最大的可用性障碍清除了。

## 几个常见的坑和应对

**1. EventSource 不支持自定义 HTTP Header**

`EventSource` 构造函数只接受一个 URL 字符串，不支持传 Header。这意味着你没法直接加 `Authorization: Bearer xxx` 做鉴权。

两种解法：要么把 token 放在 URL 参数里（`/events?token=xxx`）——但这会把 token 暴露在 URL 里，不够安全；要么放弃 `EventSource`，改用 `fetch` + `ReadableStream` 手动解析 SSE 流。

```javascript
// 支持自定义 Header 的 SSE 客户端（手动解析）
const response = await fetch('/events', {
  headers: { 'Authorization': 'Bearer ' + token }
})
const reader = response.body.getReader()
const decoder = new TextDecoder()
let buffer = ''

while (true) {
  const { done, value } = await reader.read()
  if (done) break

  buffer += decoder.decode(value, { stream: true })
  const parts = buffer.split('\n\n')
  buffer = parts.pop() // 最后一个可能不完整，留着等下一次

  for (const part of parts) {
    const match = part.match(/^data: (.+)$/m)
    if (match) {
      console.log('收到:', JSON.parse(match[1]))
    }
  }
}
```

代价是自己失去了 `EventSource` 的自动重连和 `Last-Event-ID`，需要手写重连逻辑。不过 npm 上有个 `@microsoft/fetch-event-source` 包封装好了这些，不想造轮子可以直接用。

**2. 不支持二进制数据**

SSE 只传 UTF-8 文本。需要传二进制（比如图片、文件）的话，Base64 编码是一种方案，但会增加约 33% 的体积。更好的做法是 SSE 只推元数据和资源 URL，客户端拿到 URL 后再用普通 HTTP 请求去下载。

**3. 代理和负载均衡器的缓冲**

除了前面说的 Nginx `proxy_buffering`，CDN 层也经常是个坑。Cloudflare 默认会缓冲响应，需要开启 "Real-Time Communication" 或者在 Worker 层面处理。部署到生产之前，顺着请求链路排查一遍所有中间件的缓冲策略，省得调试半天发现是 CDN 在作妖。

## 写在最后

SSE 不是什么新技术，但它足够简单、足够可靠。一个普通的 HTTP 服务加上几十行代码就能跑起来，不需要额外的协议升级、不需要专门的中间件支持、不需要谈 ws 握手——这些"简单"在生产环境里就是"少出问题"。

大部分实时推送场景根本不需要 WebSocket 的全双工能力。服务器推数据给客户端，SSE 就是最合适的选择。当你的需求是"把服务器上的变化告诉前端"，SSE 足以胜任。

如果你的架构已经在用 HTTP/2，那 SSE 的体验基本等同于 WebSocket 的单向版本，还自带重连和事件 ID 追踪。除非你真的需要双向通信或者二进制帧，否则没必要引入 WebSocket 的复杂度。

说到底，选型的原则不是"哪个更强"，而是"哪个刚好够用"。