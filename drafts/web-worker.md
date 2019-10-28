## Worker 概述

JavaScript 语言采用的是单线程模型，而 worker 可以在浏览器主线程之外的单独的线程上运行脚本。

Worker 分为三种 -- Web Worker、Service Worker 以及 Worklet，其中 Web Worker 又分为专用线程（Dedicated Worker）以及共享线程（Shared Worker）。专用线程仅仅能被生成它的脚本使用，而共享线程可以同时被多个页面（同源）的脚本共享。

Worker 拥有自己的全局作用域`WorkerGlobalScope`接口。根据 Worker 的类型不同，全局作用域具体化为`DedicatedWorkerGlobalScope`、`SharedWorkerGlobalScope`以及`ServiceWorkerGlobalScope`，且 Worker 中的关键字`self`指向这些全局作用域。

Worker 的事件循环机制也有所不同，任务队列仅包含事件（events）、回调（callbacks）以及网络活动（networking activity）。每一种 Worker 的全局作用域对象（比如`DedicatedWorkerGlobalScope`）都有一个`closing`，当这个标志设为 true 时，任务队列将丢弃之后试图加入任务队列的任务，队列中已经存在的任务不受影响（除非另有说明）。更多可以查看 [worker event loop](https://html.spec.whatwg.org/multipage/workers.html#worker-event-loop)

Worker 线程无法读取所在网页的 DOM 对象，也无法使用 `window`、`document`、`parent` 对象。但可以通过`postmessage`定义一些规则，在主线程使用达到间接使用的目的。至于 Worker 线程中可以使用哪些对象，详见 MDN[此链接](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API/Functions_and_classes_available_to_workers)

Worker 的数据传输存在两种数据传递模式--复制模式以及转移模式。复制模式使用 [结构化克隆算法](https://developer.mozilla.org/zh-CN/docs/Web/Guide/API/DOM/The_structured_clone_algorithm) ，此算法有一定的缺陷，比如不支持 DOM 对象、Function 对象以及 Error 对象的复制。转移模式支持可转让对象（ Transferable objects），可转让对象从一个上下文转移到另一个上下文而不会经过任何拷贝操作，这意味着当传递大数据时会获得极大的性能提升。`postMessage`仅支持 `MessagePort`以及`ArrayBuffer`，且转移后原来的上下文无法再获取该数据。

```js
var uInt8Array = new Uint8Array(1024*1024*32); // 32MB
for (var i = 0; i < uInt8Array .length; ++i) {
  uInt8Array[i] = i;
}
// postMessage(aMessage, transferList)
worker.postMessage(uInt8Array.buffer, [uInt8Array.buffer]); // 转移
```

## Web Worker

Web Worker 允许主线程创建 Worker 线程从而创造多线程环境，将一些任务分配给后者运行。在主线程运行的同时，Worker 线程在后台运行，两者互不干扰。

特别说明：一般情况下，Web Worker 指的是 Web Worker 中的专用线程。但本文严格区分，Web Worker 用于泛指，指的是专用线程以及共享线程。

### 使用场景

Web Worker 的意义在于可以将一些耗时的数据处理操作（CPU 密集型任务）从主线程中剥离，使主线程更加专注于页面渲染和交互。

- 文本分析
- 大数据处理
- 复杂的数学运算
- 图像处理
- ...

### 基本使用

_在使用的过程中注意区分，`new Worker()`或`new SharedWorker()`实例化出来的属性以及相对应的`DedicatedWorkerGlobalScope`和`SharedWorkerGlobalScope`的属性_

#### 特性检测

主线程可以使用`window.Worker`以及`window.SharedWorker`是否存在来分别判断是否支持专用线程以及共享线程

```js
if(window.Worker){}
if(window.SharedWorker){}
```

#### 创建

主线程可以使用`new Worker(aURL, options)`或`new SharedWorker(aURL, options)`来实例化一个 Worker 线程对象。`aURL`是一个单独的 JS 脚本文件，也可以载入 JS 代码文本的 blob 对象，但必须同源。Worker 线程无法读取本地文件，加载脚本必须来自网络。`options`是一个对象，常用的是`name`属性，对调试非常有用

在`new Worker(aURL, options)`或`new SharedWorker(aURL, options)`实例化的时候可能会因为不同源或 URL 非法使得无法启动 Worker 线程，而抛出`SecurityError`，也有可能会因为 MIME 类型不为`text/javascript`而抛出`NetworkError`，也有可能因为`aURL`无法被解析而抛出`SyntaxError`

```js
// 文件形式
var worker = new Worker('worker.js', { name: 'myDedicatedWorker' })
// 字符串拼接的 blob 形式
var myTask = `
    var i = 0;
    function timedCount(){
        i = i+1;
        postMessage(i);
        setTimeout(timedCount, 1000);
    }
    timedCount();
`;
var blob = new Blob([myTask], {type: "text/javascript"});
var myWorker = new Worker(window.URL.createObjectURL(blob))
```

```html
<!-- 内嵌 HTML 中的 blob 形式，也称「嵌入式 worker」 -->
<script id="worker" type="javascript/worker">
<!-- 这段代码不会被 JS 引擎直接解析，因为类型是 'javascript/worker' -->
<!-- 代码逻辑 -->
</script>
<script>
    var workerScript = document.querySelector('#worker').textContent
    var blob = new Blob([workerScript], {type: "text/javascript"})
    var worker = new Worker(window.URL.createObjectURL(blob))
</script>

```

#### 消息传递

Worker 线程与主线程不在同一个上下文环境，不能直接通信，需要借助`postMessage`以及`onmessage`。

Worker 线程与主线程之间通过`postMessage`进行发送消息，使用`onmessage`事件处理函数来响应消息，其中`postMessage`传递的数据可以在`onmessage`事件处理函数的`event.data`可以获取。在主线程中使用时，`onmessage`和`postMessage()`必须挂在`worker`对象上，而在 Worker 线程中使用时不用这样做。原因是在 Worker 线程内部`worker`是有效的全局作用域。且 Worker 线程内部的`self`、`this`都能表示子线程全局对象，所以 Worker 线程中`postMessage()`、`this.postMessage()`以及`self.postMessage()`是等同的。

```js
// 主线程
var myWorker = new Worker('worker.js');
var first = document.querySelector('#first')
var second = document.querySelector('#second')
var result = document.querySelector('#result')
first.onchange = function() {
  myWorker.postMessage([first.value,second.value]); // 拼接成数组
  console.log('Message posted to worker');
}
second.onchange = function() {
  myWorker.postMessage([first.value,second.value]); // 拼接成数组
  console.log('Message posted to worker');
}
myWorker.onmessage = function(e) {
  result.textContent = e.data;
  console.log('Message received from worker');
}
// 子线程
onmessage = function(e) {
  console.log('Message received from main script');
  var workerResult = 'Result: ' + (e.data[0] * e.data[1]);
  console.log('Posting message back to main script');
  postMessage(workerResult);
}
```

需要注意的是，共享线程的数据传输处理与主线程不同。主线程通过`MessagePort`访问专用线程和共享线程。专用线程的`port`会在线程创建时自动设置，并且不会暴露出来，所以并不需要我们手动设置。但是**与专用线程不同的是，共享线程与主线程通信必须通过端口对象**，且共享线程在传递消息之前，端口必须处于打开状态。

`SharedWorker`对象上有`port`属性（返回`MessagePort`对象）以及继承自`AbstractWorker`的`onerror`属性（当然还有继承自`EventTarget`的`addEventListener`等），而`onmessage`、`postMessage`等属性或方法需要借助`MessagePort`对象。

如果是共享线程使用`onmessage`进行事件监听，则不需要`MessagePort`对象的`start()`方法；如果共享线程使用`addEventListener`进行事件监听，则需要`MessagePort`对象的`start()`方法。

需要注意一下下面例子中的处理：

```js
// 共享线程的数据传输

// 主线程
var first = document.querySelector('#number1');
var second = document.querySelector('#number2');

var result1 = document.querySelector('.result1');

if (!!window.SharedWorker) {
  var myWorker = new SharedWorker("worker.js");

  first.onchange = function() {
    myWorker.port.postMessage([first.value, second.value]);
    console.log('Message posted to worker');
  }

  second.onchange = function() {
    myWorker.port.postMessage([first.value, second.value]);
    console.log('Message posted to worker');
  }

  // 共享线程需要 port 属性（返回`MessagePort`对象）
  // 但是不需要`start()`方法，因为`ommessage`已经帮忙处理
  myWorker.port.onmessage = function(e) {
    result1.textContent = e.data;
    console.log('Message received from worker');
    console.log(e.lastEventId);
  }
  // myWorker.port.addEventListener('message', function(e) {
  //     // 业务逻辑
  // }, false)
  // myWorker.port.start() // 需要显式打开

}
// 共享线程
// 使用 onconnect 连接到主线程的相同端口进行监听
onconnect = function(e) {
  var port = e.ports[0]; // 获取相关联的端口

  // 数据传输的监听
  port.addEventListener('message', function(e) {
    var workerResult = 'Result: ' + (e.data[0] * e.data[1]);
    port.postMessage(workerResult);
  });

  port.start(); // 因为调用的是 addEventListener，显示调用 start()
}
```

`postMessage()` 一次只能发送一个对象， 如果需要发送多个参数可以将参数包装为数组或对象再进行传递。Worker 的数据传输存在两种传递模式--使用 [结构化克隆算法](https://developer.mozilla.org/zh-CN/docs/Web/Guide/API/DOM/The_structured_clone_algorithm) 的复制模式以及仅支持`ArrayBuffer`的转移模式。

需要注意的是，Web Worker 的运行不会影响主线程，但与主线程交互时仍受到主线程单线程的瓶颈制约。换言之，如果 Worker 线程频繁与主线程进行交互或需要克隆与序列化数据耗时过长，主线程由于需要处理交互，仍有可能使页面发生阻塞。

#### 关闭 Web Worker

Worker 线程关闭，对于主线程而言使用`worker.terminate()`，对于 Worker 线程自身而言使用`self.close()`，但推荐使用`self.close()`以防意外关闭正在运行的 Worker 线程。需要注意的是 Shared Worker 对象及其 port 属性都没有`terminate`方法，无法使用`worker.terminate()`只能在 Worker 线程中使用`self.close()`

#### 错误处理

对于专用线程而言，主线程或专用线程可以使用`onerror`监听 Worker 线程是否发生错误。主线程中的`worker.onmessageerror`以及 Worker 线程中的`self.onmessageerror`可以监听发送数据无法序列化时的错误。

对于共享线程而言，`onerror`事件处理是在`SharedWorker`对象中，所以可以在主线程使用`worker.onerror`共享线程中使用`self.onerror`。但是`onmessageerror`是`MessagePort`对象中的事件处理，所以主线程中使用`worker.port.onmessageerror`，共享线程中使用`self.port.onmessageerror`

```js
// 主线程
worker.onerror = function () {}
// 主线程使用专用线程
worker.onmessageerror = function () {}
// 主线程使用共享线程
worker.port.onmessageerror = function () {}

// 专用线程以及共享线程
onerror = function () {}
// 专用线程
onmessageerror = function(){}
// 共享线程
onconnect = function(e) {
  var port = e.ports[0]; // 获取相关联的端口
  port.onmessageerror = function(){}
}

```

#### 加载外部脚本

Worker 线程内部加载其他脚本，需要使用`importScripts(a[, b...])`，其中`a`、`b`为脚本路径。脚本的下载顺序不固定，但执行时会按照传入 `importScripts()` 中的文件名顺序进行。这个过程是同步完成的；直到所有脚本都下载并运行完毕， importScripts() 才会返回。

#### 内容安全策略

Worker 有自己的执行上下文，所以 Worker 线程的脚本并不受限于 document 或其父级 Worker 的内容安全策略。为了给 Worker 指定内容安全策略，必须为发送 Worker 代码的请求加上相关 CSP 头部。但有一个例外，如果 worker 脚本的源是全局唯一标示符（比如 data 协议的 URL 或 blob），worker 脚本则会继承创建它的父级 Worker 或 document 的 CSP

### 更优雅的使用

为了更加方便地在项目中使用 Web Worker，可以借助一些工具。这里介绍几款常见的工具。

- worker-loader：webpack 官方出品的 loader，将 worker 当作成模块处理。在`babel-loader`的支持下，可以在 worker 脚本中使用 ES6 模块，比如`import _ from 'lodase'`。写 worker 模块就像写普通模块一样轻松。loader 本身支持多项配置，除了常见的`publicPath`以及`name`，还支持是否使用 blob 对象的`inline`（可以解决跨域问题）以及是否支持非 worker 环境的回退等。要求 Webpack v4。
- worker-plugin: Google 出品的插件，作用与`worker-loader`差不多，但是好像目前并不支持`inline`的配置
- workerize + workerize-loader：`workerize` 可以将模块移入 Worker 当中，并且将导出的方法映射成异步方法作为实例化对象的相应属性（Moves a module into a Web Worker, automatically reflecting exported functions as asynchronous proxies.)。因为`workerize(code)`接收字符串，所以单独使用并不方便。所以受`worker-loader`的启发，开发了`workerize-loader`，在达到`worker-loader`的效果上新增了导出方法的`async-await`语法糖。
- greenlet: 可以将异步方法移入 Worker 线程中，是`workerize`的单函数版本（Move an async function into its own thread.A simplified single-function version of workerize）。从语法以及简单易用上，`greenlet`比`workerize`更胜一筹。
- comlink + comlink-loader: `comlink`可以将模块移入 Worker 当中，并且可以导出几乎任意接口的数据（对象、类以及方法等等），算是`workerize`的一种**增强**（move a module with any interface into a thread）。`comlink-loader`帮忙简化了`comlink`的 API 使用，

`workerize`、`greenlet`以及`comlink`都算一种 worker wrapper。用途如同 comlink 文档所说，将基于消息的 API（比如`postMessage`以及`onmessage`的通信机制）转换为对开发者更加友好的 RPC 实现：在一个线程中使用另外一个线程的变量，就如同使用本地变量一样。

`comlink`搭配`comlink-loader`，与`workerize`搭配`workerize-loader`的使用很类似，但两者有一些差异：
- `workerize`的实现更加小巧和简洁
- `workerize`的兼容性较好，因为只用到 Promise，基本无需 polyfill。而 `comlink` 借助了 Proxy，需要引入`proxy-polyfill`
- `workerize`只支持复制模式的数据传输，而`comlink`支持复制模式以及转移模式的数据传输

```js
// index.js -- 单纯使用 workerize
let worker = workerize(`
	export function add(a, b) {
		// block for half a second to demonstrate asynchronicity
		let start = Date.now();
		while (Date.now()-start < 500);
		return a + b;
	}
`);

(async () => {
	console.log('3 + 9 = ', await worker.add(3, 9));
	console.log('1 + 2 = ', await worker.add(1, 2));
})();

```
```js
// workerize + workerize-loader 搭配使用
// worker.js
export function expensive(time) {
    let start = Date.now(),
        count = 0
    while (Date.now() - start < time) count++
    return count
}
// index.js
import worker from 'workerize-loader!./worker'

let instance = worker()  // `new` is optional

instance.expensive(1000).then(count => {
    console.log(`Ran ${count} loops`)
})
```

```js
import greenlet from 'greenlet'

let getName = greenlet( async username => {
    let url = `https://api.github.com/users/${username}`
    let res = await fetch(url)
    let profile = await res.json()
    return profile.name
})

console.log(await getName('developit'))
```

## Service Worker

详情见相关 anki 卡片
