### 简单轮询

简单轮询是在特定的时间间隔内，由浏览器对服务器发起 HTTP 请求，从而获得最新的消息或结果。

```js
function polling() {
    fetch(url).then(data => {
        process(data);
        return;
    }).catch(err => {
        return;
    }).then(() => {
        setTimeout(polling, 5000);
    });
}

polling();

```

优点：兼容性好，实现简单

缺点：占用较多的内存资源和带宽。在数据实时性上，非实时且延迟取决于请求间隔。轮询间隔难以抉择，间隔过长会导致用户不能及时接收到更新的数据；过短会导致查询请求过多，增加服务器端的负担

### 长轮询

客户端发送请求后，服务器阻塞请求，等到有数据更新或连接超时才返回。服务器端返回后客户端再次发起请求，重新建立连接。

```js
// client.js

function query() {
    fetch(url)
        .then(function(data) {
            // 请求结束，触发事件通知 eventbus
            eventbus.trigger('fetch-end', {data, status: 0});
        });
}

eventbus.on('fetch-end', function (result) {
    // 处理服务端返回的数据
    process(result);
    // 再次发起请求
    query();
});

```

```js
// server.js
const app = http.createServer((req, res) => {
    // 返回数据的方法
    const longPollingSend = data => {
        res.end(data);
    };

    // 当有数据更新时，服务端“推送”数据给客户端
    EVENT.addListener(MSG_POST, longPollingSend);

    req.socket.on('close', () => {
        console.log('long polling socket close');
        // 注意在连接关闭时移除监听，避免内存泄露
        EVENT.removeListener(MSG_POST, longPollingSend);
    });
});

```

优点：比简单轮询节约带宽；兼容性强

缺点：依然会占用较多的内存资源和带宽；数据实时性上同短轮询；相对于简单轮询，需要对服务端代码进行一定改造

### SSE

`Server-Sent Events`是 HTML5 标准的一部分。HTTP 响应内容有一种特殊的`content-type —— text/event-stream`，该响应头标识了响应内容为事件流，客户端不会关闭连接，而是等待服务端不断得发送响应结果。

```js
// client.js
const evtSource = new EventSource("ssedemo.php");
// const evtSource = new EventSource("//api.example.com/ssedemo.php", { withCredentials: true }); // 跨域的情况

// 默认的事件，用于监听没有 event 字段的消息
source.addEventListener('message', function (e) {
    console.log(e.data);
}, false);

// 用户自定义的事件名
source.addEventListener('my_msg', function (e) {
    process(e.data);
}, false);

// 监听连接打开
source.addEventListener('open', function (e) {
    console.log('open sse');
}, false);

// 监听错误
source.addEventListener('error', function (e) {
    console.log('error');
});

```

每次发送的消息由多个 message 组成，每个 message 之间使用`\n\n`分割。每个 message 由若干行组成，行的字段可取值为以下 4 个加上空字段：

- event: 事件类型。如果没有此字段，触发 onmessage 事件处理函数
- data: 数据字段。如果多个此字段，则用换行符连接作为单个值
- id: 事件 id。一旦连接断线，浏览器会使用 Last-Event-ID 头信息，将这个值发送回来，用来帮助服务器端重建连接。如果可以接受消息丢失，可以不实现 id 的生成逻辑以及该字段。如果需要恢复以及重传，则需要服务器实现一定的 id 生成机制，用于数据与 id 挂钩。更多消息可查看 [文章](https://hpbn.co/server-sent-events-sse/#esid43)
- retry: 重新连接的毫秒数。如果不是整数，将被忽略。规范建议浏览器实现的默认值为 2-3s，不同浏览器实现有可能不同。
- 当字段为空（以冒号开头），表示发送注释。使用场景是一段时间发送注释，保持连接不断

```js
// server.js

const app = http.createServer((req, res) => {
    const sseSend = data => {
        res.write('retry:10000\n');
        res.write('event:my_msg\n');
        // 文本数据传输
        res.write(`data:${JSON.stringify(data)}\n\n`);
    };

    // 注意设置响应头的 content-type
    res.setHeader('content-type', 'text/event-stream');
    // 一般不会缓存 SSE 数据
    res.setHeader('cache-control', 'no-cache');
    res.setHeader('connection', 'keep-alive');
    res.statusCode = 200;

    res.write('retry:10000\n');
    res.write('event:my_msg\n\n');

    EVENT.addListener(MSG_POST, sseSend);

    req.socket.on('close', () => {
        console.log('sse socket close');
        EVENT.removeListener(MSG_POST, sseSend);
    });
});

```

优点：实现简单；默认支持断线重连；轻量级；

缺点：IE 不支持，兼容性较差；非实时，有一定延迟（可自定义，默认 3s）

### WebSocket

WebSocket 是一个全双工的协议，适用于需要进行复杂的双向数据通讯的场景。类似于 HTTP 和 HTTPS，ws 相对应的也有基于 TLS 的 wss 用以建立安全连接。

WebSocket 的连接过程需要借助 HTTP 请求进行协议升级，升级过程这里稍微展开：

```plain
// 开始建立连接

// HTTP 请求
Connection: Upgrade // 表示要升级协议
Sec-WebSocket-Extensions: permessage-deflate; client_max_window_bits
Sec-WebSocket-Key: 0lUPSzKT2YoUlxtmXvdp+w==
Sec-WebSocket-Version: 13
Upgrade: websocket // 表示升级到 websocket 协议

// HTTP 响应
Connection: Upgrade
Origin: http://127.0.0.1:8080
Sec-WebSocket-Accept: 3NOOJEzyscVfEf0q14gkMrpV20Q=
Upgrade: websocket

```

上面的请求头中有一个`Sec-WebSocket-Key`，与响应头中的`Sec-WebSocket-Accept`是配套的。最主要的作用是来验证服务器是否真的正确“理解”了 WebSocket、该 WebSocket 连接是否有效。两者的关系用伪代码进行表示则为：

```plain
magicStr = '258EAFA5-E914-47DA-95CA-C5AB0DC85B11'
Sec-WebSocket-Accept = toBase64(sha1(Sec-WebSocket-Key + magicStr))
```

表示先进行字符串拼接，然后使用 SHA1 算法计算得到摘要，再转换为 base64 编码。

```js
// client.js
var ws = new WebSocket("wss://echo.websocket.org");

ws.onopen = function(evt) {
  console.log("Connection open ...");
  ws.send("Hello WebSockets!"); // 发送数据
};

ws.onmessage = function(evt) {
  console.log( "Received Message: " + evt.data);
  ws.close(); // 关闭连接
};

ws.onclose = function(evt) {
  console.log("Connection closed.");
};
```

关于服务端 WebSocket 的实现，因为有很多细节需要处理，所以推荐使用成熟的第三方库，比如`socket.io`。具体使用参照第三方库的使用文档。下边以`socket.io`为例：

```js
// server.js
const server = require('http').createServer();
const io = require('socket.io')(server);
io.on('connection', client => {
  client.on('event', data => { /* … */ });
  client.on('disconnect', () => { /* … */ });
});
server.listen(3000);
```

优点：数据实时；全双工通信；没有同源限制，客户端可以与任意服务器通信；可以发送文本，也可以发送二进制

缺点：兼容性较差；开发难度较大

### SSE VS WebSocket

- 协议上，SSE 使用 HTTP 协议，而 WebSocket 使用基于 TCP 的另一应用层协议
- SSE 轻量级，WebSocket 协议相对复杂
- SSE 默认支持断线重连，WebSocket 需要自己实现
- SSE 一般用于传输文本，二进制数据需要编码后传输；WebSocket 默认支持传输二进制数据
- SSE 支持自定义发送的消息类型
- 目前而言（2019.10），WebSocket 的兼容性较 SSE 兼容性稍好。WebSocket 支持 IE10 而 SSE 不支持 IE。
- SSE 跨域的情况需要使用`withCredentials`选项表示是否发送 cookie，而 WebSocket 默认支持跨域
- SSE 单向通信，只能服务器端推送到客户端；WebSocket 是全双工通信，支持服务器与客户端消息互推
