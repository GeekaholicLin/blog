对于 JS 而言，异常的出现不会直接导致 JS 引擎崩溃，最多只会使当前执行的任务终止。

### 代码块级别

`try...catch`代码块只会捕获同步代码，不能捕获异步代码，原因可以从 _JS 的执行机制与任务队列方面进行分析_，执行上下文发生变化从而无法捕捉。想捕获异步代码的异常只能在异步逻辑内部进行捕获。异步代码包括但不局限于`Promise`、`setTimeout`。

`try...catch`也不能获取语法错误

Promise 的**构造函数**，以及被 `then`调用执行的函数基本上都可以认为是在 `try...catch` 代码块中执行的。并且在`catch`子句当中会调用`onRejected`的回调，也就是会被 Promise 的`catch`方法捕获。也可以使用`async...await`的语法，使得 Promise 的`try...catch`代码块正常工作。

从 Babel 编译后的代码来看，`async...await`能正常地被`try-catch`捕获异常的原因，主要是未完成时`Promise.resolve(value).then(_next, _throw)`。当`asyncFunc`的 promise 中出现异常或 reject 拒绝时，相当于执行了`_throw`，调用`generator`对象的`thow`方法，所以会抛出异常进入`catch`子句。

```js
var asyncFunc = () =>
  new Promise((resolve, reject) => {
    setTimeout(() => {
      // 模拟请求
      reject("Error!");
    }, 1000);
  });

async function main() {
  try {
    const res = await asyncFunc();
    console.log(res);
  } catch (e) {
    console.error(e);
  }
}
```

#### 优化

为了在使用 await 的时候减少`try...catch`的样板代码，可以封装处理方法，类似 [await-to-js](https://github.com/scopsy/await-to-js/blob/master/src/await-to-js.ts)，对异步函数进行包装并处理异常，把异常和结果作为二元组返回。参照`await-to-js`的源码，为了更上一层楼，可以**在 promise 中`reject`自定义异常**而不需要`errorExt`。并且为了异常的通用处理，可以传入相同的异常处理方法`errorHandler`而不需要重复进行`if(error){}`的判断是否出错（当然引入了`errorHandler`后依然可以使用`if(error){}`进行进一步的处理）

```js
// 对`await-to-js`中的 to 进行改造
function to(promise, errorHandler) {
  return promise
    .then(function(data) {
      return [null, data];
    })
    .catch(function(err) {
      if (errorHandler) errorHandler(err);
      return [err, undefined];
    });
}

// 自定义异常
class FetchError extends Error {
  constructor({ code, message }) {
    super(message);
    this.code = code || 200000;
  }
}
// 异常处理方法
function fetchErrorHandler(err) {
  if (err instanceof FetchError) {
    console.log("处理异常");
  }
}
// 模拟出错的 promise
var asyncFunc = () =>
  new Promise((resolve, reject) => {
    setTimeout(() => {
      // 模拟请求
      reject(new FetchError({ message: "fetch_error" }));
    }, 1000);
  });
async function main() {
  const [err, res] = await to(asyncFunc(), fetchErrorHandler);
  if (err) {
    // 除了 fetchErrorHandler 的通用处理，还可以有其他特别处理
    console.log("做一些特别的处理");
    return;
  }
}
```

### 全局处理

#### window.onerror 与 window.addEventListener('error')

> 在实际的使用过程中，`onerror` 主要是来捕获预料之外的错误，而`try-catch`则是用来在可预见情况下监控特定的错误，两者结合使用更加高效

当 JS 运行时错误发生时，window 会触发一个`ErrorEvent`接口的 error 事件，并执行`window.onerror()`

`window.addEventListener('error', function(event){})`虽然效果类似，但`window.onerror()`的兼容性更好，且获得的错误信息更加详细。

需要注意的是因为历史原因`window.onerror(message, source, lineno, colno, error)`的参数和`element.onerror(event)`的参数不一样。

优点：可以捕获异步异常

缺点：

- 无法捕获语法错误
- 无法处理静态资源加载失败的异常。当一项资源（如`<img>`或`<script>`）加载失败，加载资源的元素会触发一个 Event 接口的 error 事件，并执行该元素上的`onerror()`处理函数。_这些 error 事件不会向上冒泡到 window_。

对于静态资源加载失败的异常可以使用`window.addEventListener('error', function(event){})`进行解决。

```js
window.onerror = function(message, source, lineno, colno, error) {
// message：错误信息（字符串）。
// source：发生错误的脚本 URL（字符串）
// lineno：发生错误的行号（数字）
// colno：发生错误的列号（数字）
// error：Error 对象（对象）
console.log('捕获到异常：',{message, source, lineno, colno, error});
}
Jartto; // 未定义的变量
Jartto‘; // Uncaught SyntaxError，语法错误
```

注意事项：

- `window.onerror` 最好写在所有 JS 脚本的前面
- 例子不可以在 Chrome 控制台直接使用
- `window.onerror` 函数只有在返回 `true` 的时候，异常才不会向上抛出
- 当使用`window.addEventListener`时注意不要和`window.onerror`进行重复处理，建议只有`event.target`是`HTMLScriptElement`、`HTMLLinkElement`或`HTMLImageElement`实例的时候才在`window.addEventListener`中上报异常。

#### unhandledrejection

为了防止有漏掉的 Promise 异常，建议在全局增加一个对`unhandledrejection`的监听。

```js
window.addEventListener("unhandledrejection", function(e) {
  e.preventDefault(); // 阻止冒泡，让控制台不显示错误
  console.log("捕获到异常：", e);
});
Promise.reject("promise error");
```

### 特殊场景

#### iframe

同域 iframe 的异常需要使用`window.onerror`。不同域且非第三方（开发者拥有控制权）的情况可以借助跨域通信手段，将错误信息传递出来。

```js
window.frames[0].onerror = function(message, source, lineno, colno, error) {
  console.log("捕获到 iframe 异常：", {
    message,
    source,
    lineno,
    colno,
    error
  });
  return true;
};
```

#### 跨域脚本

在`script`标签引入不同域的脚本文件，当脚本文件报错时，引用页面只会抛出`Script Error`的错误。如果想获得详细的错误信息，需要配置`script`标签的`crossorigin`属性，同时被引入脚本的 HTTP 响应头部`Access-Control-Allow-Origin`需要包含当前引用域。

除了上诉方法，还有解决`Script Error`的 [另类思路](https://juejin.im/post/5c00a405f265da610e7fd024)，但是受用面窄且需要改动过多，这里仅用于思维拓展，_不建议用于正式环境_。

基本思路是劫持相关的原型方法，利用包装进行进一步的逻辑处理。在包装逻辑中，调用`try...catch`原有的回调方法，_重新 throw 出来的异常是同域代码_，所以可以被引用页面的`window.onerror`捕获而不丢失堆栈信息。

但是劫持原型方法本来就是一种坏味道，而且这里可能要劫持很多个原型方法，所以此方法并不推荐。

这里以`EventTarget.prototype.addEventListener`进行举例说明。

```js
// index.html
const originAddEventListener = EventTarget.prototype.addEventListener;
EventTarget.prototype.addEventListener = function(type, listener, options) {
  const addStack = new Error(`Event (${type})`).stack; // 用于异常发生时，当前信息加入错误堆栈
  const wrappedListener = function(...args) {
    try {
      // 核心所在，将原有的 listener 包裹在 try...catch 中
      return listener.apply(this, args);
    } catch (err) {
      err.stack += "\n" + addStack;
      throw err;
    }
  };
  return originAddEventListener.call(this, type, wrappedListener, options);
};
// iframe.html
const btn4k = document.querySelector("#btn-4000");
btn4k.addEventListener("click", () => {
  throw new Error("Fail 4000");
});
```

#### React

React 16 引入的错误边界`Error Boundary`是为了捕获 UI 渲染过程中的异常，防止破坏整个程序。大多数情况下我们可以在整个程序中定义一个 error boundary 组件，之后就可以一直使用它。

更多详情请见 [官方文档](https://reactjs.org/docs/error-boundaries.html)

#### Koa

koa 中的错误处理相关：[Error-Handling](https://github.com/koajs/koa/wiki/Error-Handling) 以及 [Koa 中的错误处理](https://www.cnblogs.com/Wayou/p/error_handling_in_koajs.html)

### 总结

- `try...catch`可捕获同步异常，无法捕获异步异常以及语法错误
- `window.onerror`可捕获同步与异步异常，无法捕获静态资源错误与语法错误
- `window.addEventListener('error')`可捕获同步、异步以及静态资源异常，无法捕获语法错误。建议只有`event.target`是`HTMLScriptElement`、`HTMLLinkElement`或`HTMLImageElement`实例的时候才在`window.addEventListener`中上报异常，但要注意重复上报问题
- `window.addEventListener('unhandledrejection')`处理未被 catch 的 promise 异常

需要注意的是异步异常的范围，XHR 对象的异常由`xhr.addEventListener('error')`进行处理，无法被`window.onerror`或`window.addEventListener('error')`进行捕获

### 拓展

#### 异常的监控和上报

异常监控的意义十分重大，它可以让我们及时发现问题所在，比如性能问题、JS 代码逻辑问题等，便于优化项目。也可以通过报错信息，远程定位问题所在，从而解决无法复现问题。

至于怎么监控异常、上报异常、监控异常指标的确定以及在页面崩溃的时候上报数据，那就是另外一个话题了。
