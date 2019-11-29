## 基础问题

### 对 Webpack 的认知

Webpack 是一个可高度配置的现代 JavaScript 应用程序模块打包器（module bundler），会根据入口递归地构建依赖关系图，并使用 loader 处理不同文件和 plugin 拓展 webpack 的功能，最后生成一个或多个 bundle

### 与 Grunt 或 Gulp 的不同

三者都是前端构建工具

grunt 和 gulp 是基于任务和流的。找到一个（或一类）文件，对其做一系列链式操作，更新流上的数据，整条链式操作构成了一个任务，多个任务就构成了整个 web 的构建流程

webpack 是基于入口的，是一个可高度配置的现代 JavaScript 应用程序模块打包器（module bundler），会根据入口递归地构建依赖关系图，并使用 loader 处理不同文件和 plugin 拓展 webpack 的功能，最后生成一个或多个 bundle。

### Webpack 的构建流程

1. 初始化参数：从配置文件和 Shell 语句中读取与合并参数，得出最终的参数
2. 开始编译：用上一步得到的参数初始化 Compiler 对象，加载所有配置的 plugin，执行对象的 run 方法开始执行编译
3. 确定入口：根据配置中的 entry 找出所有的入口文件
4. 编译模块：从入口文件出发，找出依赖的 module；调用所配置的 Loader 对（module）进行转换，再找出此 module 依赖的其他 module，再递归本步骤直到所有入口依赖的文件都经过了本步骤的处理。
5. 完成模块编译：在经过第 4 步使用 Loader 转换完所有模块后，得到了每个模块被转换后的最终内容以及它们之间的依赖关系
6. 输出资源：根据入口和模块之间的依赖关系，组装成一个个包含多个模块的 Chunk，再把每个 Chunk 转换成一个单独的文件加入到输出列表，这步是可以修改输出内容的最后机会。
7. 输出完成：在确定好输出内容后，根据配置确定输出的路径和文件名，把文件内容写入到文件系统

> 在以上过程中，Webpack 会在特定的时间点广播出特定的事件，plugin 在监听到感兴趣的事件后会执行特定的逻辑，并且 plugin 可以调用 Webpack 提供的 API 改变 Webpack 的运行结果

![超级详细的构建流程](http://image.geekaholic.cn/20191121161717.png@0.8)

### 什么是 Webpack 热更新？并说明其处理流程

webpack 的热更新（HMR，Hot Module Replacement）指对代码修改并保存后，webpack 将会对代码进行重新打包，并将改动的模块发送到浏览器端，浏览器用新的模块替换掉旧的模块，去实现局部更新页面而非整体刷新页面。

优点在于可以保存应用的状态，提高开发效率。

> 它只能与实现和理解 HMR API 的 loader 一起使用，比如`style-loader`和`react-hot-loader`。

![流程图](http://image.geekaholic.cn/20191125131055.png@0.8)

步骤为：

1. 在启动 devserver 的时候，使用`socketjs`在服务端`webpack-dev-server`和浏览器端`webpack-dev-server/client`建立 websocket 长链接，并使用 webpack api 监听 compiler 的`done`事件

```js
// webpack-dev-server/lib/Server.js
compiler.plugin('done', (stats) => {
  // stats.hash 是最新打包文件的 hash 值
  this._sendStats(this.sockets, stats.toJson(clientStats));
  this._stats = stats;
});
...
Server.prototype._sendStats = function (sockets, stats, force) {
  if (!force && stats &&
  (!stats.errors || stats.errors.length === 0) && stats.assets &&
  stats.assets.every(asset => !asset.emitted)
  ) { return this.sockWrite(sockets, 'still-ok'); }
  // 调用 sockWrite 方法将 hash 值通过 websocket 发送到浏览器端
  this.sockWrite(sockets, 'hash', stats.hash);
  if (stats.errors.length > 0) { this.sockWrite(sockets, 'errors', stats.errors); }
  else if (stats.warnings.length > 0) { this.sockWrite(sockets, 'warnings', stats.warnings); } 	  	else { this.sockWrite(sockets, 'ok'); }
};
```

1. 借助`memory-fs`，webpack 对文件系统进行 watch 打包到内存中。
2. 在文件发生改变后，webpack 重新编译，回调 webpack-dev-server 的`done`事件监听函数，然后 webpack-dev-server 将 hash 值通过 websocket 发送到浏览器端`webpack-dev-server/client`。
3. 当`webpack-dev-server/client`接收到 hash 消息的时候先暂存，随后在接收到 ok 消息，就进行 reload 操作，而 `webpack-dev-server/client`会根据 hot 配置决定 reload 操作是刷新浏览器还是 HRM

```js
// webpack-dev-server/client/index.js
hash: function msgHash(hash) {
    currentHash = hash;
},
ok: function msgOk() {
    // ...
    reloadApp();
},
// ...
function reloadApp() {
  // ...
  if (hot) {
    log.info('[WDS] App hot update...');
    const hotEmitter = require('webpack/hot/emitter');
    hotEmitter.emit('webpackHotUpdate', currentHash);
    // ...
  } else {
    log.info('[WDS] App updated. Reloading...');
    self.location.reload();
  }
}
```

![](http://image.geekaholic.cn/20191125130853.png@0.8)

4. 从上边代码可以看出，当 HRM 的时候触发`webpackHotUpdate`事件，这会使得监听了`webpackHotUpdate`的`webpack/hot/dev-server`调用`HMR runtime`中的 check 方法。
5. check 方法中使用`JSONP runtime`中的`hotDownloadManifest`和`hotDownloadUpdateChunk`方法，分别的作用是前者调用 AJAX 请求`${hash}.hot-update.json`查看是否有更新（如果有更新，json 文件中有包含更新的文件列表），而后者是根据需要更新列表拼接文件名`${moduleId}.${hash}.hot-update.js`并以 JSONP 的形式请求该文件

![](http://image.geekaholic.cn/20191125130928.png@0.8)

6. JSONP 文件请求后其实就是调用的`webpackHotUpdate`方法，找到旧的模块及其依赖并对其进行删除，接着使用`moduleId`作为 key 进行重新赋值，同时`webpack/hot/dev-server`会根据结果是否报错进行决定是否刷新浏览器作为回退方案。
7. HRM 只是更新了模块，但是不知道业务代码无法得知是否 HRM 且如何应用 HRML，所以借助其他工具比如`react-hot-loader`，在 HRM 的时候进行一定的更新操作。这也是为什么 React HRM 需要在入口处应用`hot(module)(App)`进行改造。

本小节参考 [Webpack Hot Module Replacement 的原理解析 · Issue #15 · Jocs/jocs.github.io](https://github.com/Jocs/jocs.github.io/issues/15)

### chunk、bundle 和 module 有什么区别？

![编译截图](http://image.geekaholic.cn/20191121173444.png@0.8)

module 指的是应用程序中的`import`、`require()`、`@import`、`url()`或`<img src=...>`等形式引入的各式各样的文件，而不仅仅是 JS 文件（因为非 JS 或 JSON 文件可以使用 loader 处理）。在 Webpack 编译的时候，左边显示`Asset`，本质上就是`module`了。一定要说差别的话，`asset`常用于表述广泛含义上的「资源文件」，而`module`常用于表述 webpack 编译过程中导入的资源文件。

chunk 是 webpack 在打包过程中内部的特定术语。基本上一个文件就是一个 chunk。在 webpack 4 之后分为异步加载的 async chunk 以及同步加载的 inital chunk。在通过`splitChunks`的`cacheGroups`中的`chunks`配置聚合多个`chunks`可以变成一个新的 chunk。

bundle 就是最后输出的文件了，由多个 modules 组成，包含已进行加载和经过编译的源文件的最终版本。bundle 与 chunk 有着直接关系，但是不一定是一一对应，因为可以通过`splitChunks`的`cacheGroups`中的`chunks`配置聚合多个`chunks`变成一个新的 chunk，再输出 bundle。

### 什么是 compilation 和 compiler

compilation 对象代表某个版本的资源对应的编译进程，当你跑 webpack 的 development 中间件，每当检测到一个文件被更新之后，一个新的 comilation 对象会被创建，从而引起新的一系列的资源编译。一个 compilation 含有关于模块资源的当前状态、被编译的资源，改变的文件和监听依赖的表面信息。compilation 也提供很多回调方法，在一个插件可能选择执行制定操作的节点。

compiler 对象代表的是整个 webpack 的配置环境，这个对象只在 webpack 开始的时候构建一次，且所有的操作设置包括 options，loaders，plugin 都会被配置，当在 webpack 中应用插件时，这个插件会接受这个 compiler 对象的引用。通过 webpack 的主环境去使用这个 compiler。

compiler 是针对 webpack 的，是不变的 webpack 环境，而 compilation 这个就是每次有一个文件更新，然后会重新生成一个。

### hash、chunkhash 和 contenthash 有什么区别？

- hash：compilation 的 hash 值，跟整个项目的构建相关，只要项目里有文件更改，整个项目构建的 hash 值都会更改，并且全部文件都共用相同的 hash 值
- chunkhash：chunk 的 hash 值，根据不同的入口文件 (Entry) 进行依赖文件解析、构建对应的 chunk，生成对应的哈希值。其依赖的模块更新但本身不更新，这个 chunk 也会更新。比如导入 CSS 的 JS 文件，在修改 CSS 的时候（并于之后通过插件分离成单独一个包），不仅仅 CSS 的 chunkhash 会更新，JS 的 chunkhash 也会更新，因为两者是属于同一个 chunk。相反地，只修改 JS 不修改 CSS，两者也会一起更新
- contenthash：目前推荐使用，只与输出的包的内容有关

### 如何正确缓存构建？

除了 hash 要选择 contenthash 以外，还需要对 webpack 内部的 chunk 和 module 进行正确的命名处理。因为每个 `module.id` 会默认地基于解析顺序 (resolve order) 进行增量。也就是说，当解析顺序发生变化，ID 也会随之改变，所以命名很重要（正式环境配置`moduleIds`即可，其他三项用于开发环境的调试）。

以下配置都是配置项`optimization`的子项。

#### namedModule 以及 namedChunks

以下图片来自 [文章](https://segmentfault.com/a/1190000017066322)。

`optimization.namedModules` 表示是否给 module 更有意义的名称，方便调试。开发模式默认打开，正式模式默认关闭。会应用`NamedModulesPlugin`，采用模块的路径而不是 ID 数字，但是此插件会使得构建时间变长。一般建议保持默认。

`namedModules: true`以及`namedModules: false`的对比：

![](http://image.geekaholic.cn/20191125110940.png@0.8)

![](http://image.geekaholic.cn/20191125110953.png@0.8)

`optimization.namedChunks` 表示是否给 chunk 更有意义的名称，方便调试。开发模式默认打开，正式模式默认关闭。

`namedChunks: true`以及`namedChunks: false`的对比：

![](http://image.geekaholic.cn/20191125111145.png@0.8)
![](http://image.geekaholic.cn/20191125111155.png@0.8)

#### chunkIds 以及 moduleIds

- chunkIds：告诉 webpack 选择 chunk id 的时候选用哪种方式。可以配置为`chunkIds: 'named'`便于调试。
- moduleIds: 告诉 webpack 选择 module id 的时候选用哪种方式。为了能够正确应用缓存，一般在正式环境使用`moduleIds: 'hashed'`或者`moduleIds: 'deterministic'`，分别调用了`HashedModuleIdsPlugin`和`DeterministicModuleIdsPlugin`，区别在于后者的哈希串更短（默认`maxLength: 3`）。在 webpack 5 中，正式环境模式将`moduleIds: 'deterministic'`设置为了默认。

### webpack-dev-server 和 http 服务器有什么区别？

webpack-dev-server 集成在 webpack 中，可以通过简单的配置就能与 webpack 进行很好地交互。比如可以使用内存来存储 webpack 开发环境下的打包文件，再比如可以使用模块热更新，相比传统 http 服务器开发更加简单高效

### 如何优雅地让旧项目支持 CSS Module？

借助`Rule`的`oneOf`以及`resourceQuery`，可以让特定查询尾缀字符串的导入样式进行 CSS Module 的处理。

```js
// https://github.com/css-modules/css-modules/pull/65#issuecomment-355078216
rules: [
  {
    test: /\.css$/,
    oneOf: [
      {
        use: [require.resolve("style-loader"), require.resolve("css-loader")]
      },
      {
        resourceQuery: /^\?module$/,
        use: [
          require.resolve("style-loader"),
          {
            loader: require.resolve("css-loader"),
            options: {
              importLoaders: 1,
              modules: true,
              localIdentName: "[name]__[local]___[hash:base64:5]"
            }
          },
          require("./postcss-loader")
        ]
      }
    ]
  }
];
// import './global.css?module';
```

### css-loader 和 sass-loader 的`@import`等语法问题

css-loader 对于`url()`语法不处理外部 url 以及相对于根的 url（类似`/static`，前面有`/`）语法

```text
// 相对路径语法
url(image.png) => require('./image.png')
url('image.png') => require('./image.png')
url(./image.png) => require('./image.png')
url('./image.png') => require('./image.png')
url('http://dontwritehorriblecode.com/2112.png') => require('http://dontwritehorriblecode.com/2112.png') //
image-set(url('image2x.png') 1x, url('image1x.png') 2x) => require('./image1x.png') and require('./image2x.png')
// 以下为模块语法
url(~module/image.png) => require('module/image.png')
url('~module/image.png') => require('module/image.png')
url(~aliasDirectory/image.png) => require('otherDirectory/image.png')
```

css-loader 的`@import`语法同理，只不过对于绝对路径和相对于根的语法而言直接不改变（比如`@import 'http://x.y/z.css'`或`@import '/a/b/c.css'`会真的变成 css 中的`@import`语法，在运行时执行导入）。

sass-loader 的`@import`语法是直接将模块名字`request name`传给 webpack，让 webpack 去寻找。具体表现为：相对路径和模块语法，与 css-loader 的`@import`语法大同小异；而绝对路径或相对于根的语法都是**当作本地磁盘的目录**进行查找，所以很大可能会找不到 module。

```text
@import '~bootstrap'; // 模块语法，让 webpack 从`resolve.modules`的配置中寻找，通常是`node_modules`文件夹下
@import './section.scss' // 相对路径
@import "style.scss" // 相对路径，等同上面
@import "/home/a.test" // 会当成磁盘路径进行查找
```

_而最麻烦的当属 sass-loader 的`url()`语法。_

它不提供 url 覆盖（并不是指最终结果的 url 字符串覆盖，而是**模块分析过程中**的 url 覆盖），那会导致什么问题？**可能**会导致找不到`url()`所引用的 module。

比如，在入口的`index.scss`文件处导入了其他目录下有着`url()`语法的 scss 文件`@import './subdir/inner.scss'`，也就是内嵌`@import`。因为`sass-loader`的处理是，在`index.scss`处将所有`@import`的内嵌模块打包到了一个文件里面，且不自动处理 url 覆盖，导致在`sass-loader`实际编译的时候，在`index.scss`文件中拥有了其他 scss 文件的`url()`语句，层级对应不上，所以`url()`导致引用的 module 找不到。

如果想看原话和原来的例子，请见`resolve-url-loader`的 [README](https://github.com/bholloway/resolve-url-loader/blob/master/packages/resolve-url-loader/README.md#why)

一种解决方案就是借助`resolve-url-loader`让它帮忙做 url 的覆盖，在`sass-loader`处理源文件**之后**，帮助我们处理`url()`的路径问题。

```js
rules: [
  {
    test: /\.scss$/,
    use: [
      {
        loader: "css-loader",
        options: {}
      },
      {
        loader: "resolve-url-loader",
        options: {}
      },
      {
        loader: "sass-loader",
        options: {
          sourceMap: true, // sourceMap 要在 resolve-url-loader 之前
          sourceMapContents: false
        }
      }
    ]
  }
];
```

另外一种解决方案是用模块语法，让 webpack 去帮忙展开为对应的绝对路径而不用相对路径。具体做法是`~`符号提示 webpack 这是模块语法，再借助配置中的`resolve.alias`，比如`backgroud: url(~@resource/img.png)`

## 常见的配置问题

### Webpack 配置中的 context 的作用是什么？

使用一个绝对路径作为基础目录，用于从配置中解析入口起点 (entry point) 和 loader，默认值为`process.cwd()`，但推荐设置为`src`所在目录，这可以使得 webpack 配置独立于目前工作目录 CWD。

这样`entry`的解析就是从`src`开始。

```js
module.exports = {
  //...
  entry: {
    home: "./home.js", // 入口点就是`src/home.js`
    about: "./about.js",
    contact: "./contact.js"
  }
};
```

### Webpack 配置中的 entry 和 output 的作用是什么

entry（入口起点）用于指示 webpack 应该使用哪个模块，来作为构建其内部依赖图 (dependency graph) 的开始，以便找出直接或间接依赖的模块和库。

output（输出）告诉 webpack 在哪里输出以及如何输出（包括但不限于：命名、输出包的打包方式等）它所创建的 bundle。

### Webpack 配置中的 target 和 output.libraryTarget 的区别是什么？

`target`（构建目标）告诉 Webpack 构建包所要运行的环境，使得 Webpack 针对设置的环境进行编译。因为服务器端、Web 端以及桌面端都可以使用 JS 编写且用 Webpack 打包，而每个环境有着自身特定的东西。默认值为`web`。比如说设置为`electron-main`，Webpack 会在构建的时候包含 electron 平台特定的变量。

`output.libraryTarget` 则是告诉 Webpack 用什么形式（比如`'umd'`）导出打包的 bundle，默认值为`'var'`，可和`output.library`以及`output.libraryExport`搭配使用，常见于前端库开发的场景。比如：

```js
// library 以及 libraryTarget 的使用
// 以下 `_entry_return_` 表示为入口文件（也就是打包后的 IIFE）执行后的返回值

// `library: 'MyLibrary'`且`libraryTarget: 'var'`
// 相当于将返回值赋值给 MyLibrary 变量
var MyLibrary = _entry_return_; // var MyLibrary = (function(modules) {})();

// `library: 'MyLibrary'`且`libraryTarget: 'assign'`
// 相当于重新赋值给存在的 MyLibrary 变量（如果未存在，则相当于全局；如果 library 不指定则构建失败），谨慎使用
MyLibrary = _entry_return_;

// libraryTarget 除了以上的'var'和'assign'外，下面的都是将`library`作为`libraryTarget`对象的属性。注意：如果不设置 `output.library` 将导致由入口起点返回的所有属性，都会被赋值给给定的对象；这里并不会检查现有的属性名是否存在。
// `library: 'MyLibrary'`且`libraryTarget: 'this'`
this["MyLibrary"] = _entry_return_;
// `library: 'MyLibrary'`且`libraryTarget: 'window'`
window["MyLibrary"] = _entry_return_;
// `library: 'MyLibrary'`且`libraryTarget: 'global'`
global["MyLibrary"] = _entry_return_;
// `library: 'MyLibrary'`且`libraryTarget: 'commonjs'`
exports["MyLibrary"] = _entry_return_;
require("MyLibrary").doSomething();
// 除此之外，还有模块定义系统的取值，这会使得 bundle 会被一些特定的 header 包装起来
// `library: 'MyLibrary'`且`libraryTarget: 'commonjs2'`，此时`MyLibrary`会被省略
module.exports = _entry_return_;
require("MyLibrary").doSomething();
// `library: 'MyLibrary'`且`libraryTarget: 'amd'`
define("MyLibrary", [], function() {
  return _entry_return_;
});
require(["MyLibrary"], function(MyLibrary) {
  // ...
});
// `library: 'MyLibrary'`且`libraryTarget: 'umd'`，一般使用这个
(function webpackUniversalModuleDefinition(root, factory) {
  if (typeof exports === "object" && typeof module === "object")
    module.exports = factory();
  else if (typeof define === "function" && define.amd) define([], factory);
  else if (typeof exports === "object") exports["MyLibrary"] = factory();
  else root["MyLibrary"] = factory();
})(typeof self !== "undefined" ? self : this, function() {
  return _entry_return_; // 此模块返回值，是入口 chunk 返回的值
});
// 当然可以使用不同对象或模块系统的多个不同命名：
module.exports = {
  //...
  output: {
    library: {
      root: "MyLibrary",
      amd: "my-library",
      commonjs: "my-common-library"
    },
    libraryTarget: "umd"
  }
};
```

```js
// libraryExport 的使用
// libraryExport 用于指定返回值的哪一个模块进行赋值
// `library: 'MyLibrary'`且`libraryTarget: 'this'` 且`libraryExport: 'default'`
this["MyLibrary"] = _entry_return_.default;
// 可以使用数组进行子模块赋值
// `library: 'MyLibrary'`且`libraryTarget: 'this'` 且`libraryExport: ['MyModule', 'MySubModule']`
this["MyLibrary"] = _entry_return_.MyModule.MySubModule;
```

### Webpack 的 output 配置 filename 与 chunkFilename 的区别

filename 是用于 entry chunk 文件的名称；chunkFilename 用于非入口（non-entry）chunk 文件的名称。取值的情况与 filename 相同，可以使用形如`[name]`的占位符。

### Webpack 内部命名占位符有哪些？

由 webpack 内部插件 [TemplatedPathPlugin](https://github.com/webpack/webpack/blob/master/lib/TemplatedPathPlugin.js) 提供。

- `[hash]`
- `[contenthash]`
- `[chunkhash]`
- `[name]`：模块（module）名字
- `[id]`：模块 id
- `[query]`：模块查询字符串，即模块文件名的`?`后的字符串

`[hash]`和`[chunkhash]`的长度默认为 20，可以通过全局的`output.hashDigestLength`或`[hash:16]`、`[chunkhash:16]`进行 hash 长度更改。

### webpack 配置中的 resolve 有什么用处

用于解析模块请求的配置（for resolving module requests）。

#### alias

alias 可以根据变量替换`import`或`require`路径，或者可以使用别名，简化导入路径。

```js
// platform/create-context-2d.browser.js
export function createContext2D() {
  const canvas = document.createElement("canvas");
  return canvas.getContext("2d");
}
// platform/create-context-2d.minigame.js
export function createContext2D() {
  const canvas = wx.createCanvas();
  return canvas.getContext("2d");
}
// platform/create-context-2d.miniprogram.js
export function createContext2D() {
  return wx.createCanvasContext("canvas");
}

// index.js

import Utility from "Utilities/utility"; // import Utility from 'src/utilities/utility'
import Test1 from "xyz"; // 精确匹配，所以 path/to/file.js 被解析和导入
import Test2 from "xyz/file.js"; // 非精确匹配，触发普通解析
```

```js
resolve: {
    alias: {
      'create-context-2d': `./src/platform/create-context-2d.${env.platform}.js`, // 这样不同的打包命令会导入不同路径的文件
      Utilities: path.resolve(__dirname, 'src/utilities/'),
      xyz$: path.resolve(__dirname, 'path/to/file.js')
    },
  }
```

#### modules

告诉 webpack 解析模块时应该搜索的目录，如果是相对路径则会查找当前目录或不断地往祖先路径查找，推荐使用绝对路径。

```js
module.exports = {
  //...
  resolve: {
    modules: [path.resolve(__dirname, "src"), "node_modules"]
  }
};
```

#### extensions

当请求的模块没有后缀名的时候，使用设置的顺序进行匹配。默认值为`['.wasm', '.mjs', '.js', '.json']`。

```js
module.exports = {
  //...
  resolve: {
    extensions: [".js", ".json", ".jsx", ".css"]
  }
};
```

## Loader 和 Plugin

### loader 和 plugin 的区别

webpack 只能理解 JavaScript 和 JSON 文件。loader 让 webpack 能够去处理其他类型的文件，并将它们转换为有效 模块，以供应用程序使用，以及被添加到依赖图中。

```js
module.exports = {
  module: {
    rules: [{ test: /\.txt$/, use: "raw-loader" }]
  }
};
```

表示在`import`以`.txt`结尾的文件（`test`正则匹配）时，先使用`raw-loader`进行转换。

loader 用于转换某些类型的模块，而 plugin 则可以用于执行范围更广的任务，用于拓展 webpack 的功能，包括但不限于：打包优化，资源管理，注入环境变量等。在 Webpack 运行的生命周期中会广播出许多事件，Plugin 可以监听这些事件，在合适的时机通过 Webpack 提供的 API 改变输出结果。

一条规则中的多个 loader 执行的顺序是逆序，而 plugin 是依据其自身在编译生命周期的哪个阶段进行监听实现。

### 有哪些常见的 Loader 且用处是什么

- file-loader：把文件输出到一个文件夹中，在代码中通过相对 URL 去引用输出的文件

我们都知道，webpack 最终会将各个模块打包成一个文件，因此我们样式中的 url 路径是相对入口 html 页面的，而不是相对于原始 css 文件所在的路径的。这就会导致图片引入失败。这个问题是用 file-loader 解决的，file-loader 可以解析项目中的 url 引入（不仅限于 css），根据我们的配置，将图片拷贝到相应的路径，再根据我们的配置，修改打包后文件引用路径，使之指向正确的文件。

- url-loader：和 file-loader 类似，但是能在文件很小的情况下以 base64 的方式把文件内容注入到代码中去
- source-map-loader：常用于提取提供 source-map 的第三方库内部的 source-map，以方便断点调试
- image-loader：加载并且压缩图片文件
- babel-loader：把 ES6 转换成 ES5
- css-loader：加载 CSS，支持模块化（CSS module）、文件导入（`@import`）等特性
- style-loader：把 CSS 代码注入到 DOM（内联的形式）。
- eslint-loader：通过 ESLint 检查 JavaScript 代码
- sass-resources-loader: 为每个导入的 scss 等样式文件（less 也支持）自动在头部使用`@import`

```js
// webpack.config.js
module.exports = {
  module: {
    rules: [
      test: /\/scss$/,
      use:[
        'style-loader',
        'css-loader',
        'postcss-loader',
        'sass-loader',
        {
            loader: 'sass-resources-loader',
            options: {
              resources: [
                path.resolve(__dirname, './client/styles/variable.scss'),
                path.resolve(__dirname, './client/styles/mixins.scss')]
            }
          }
      ]
    ]
  }
}

```

### 有哪些常见的 Plugin 且用处是什么

- html-webpack-plugin：在编译后，根据提高的模版文件，分析依赖关系，自动插入 bundle 等资源
- define-plugin：定义环境变量（实则是在编译的时候替换，类似「宏」）
- CleanWebpackPlugin：在配置以后，每次打包时，清空所配置的文件夹

### 简述如何编写 Loader

编写 `Loader` 时要遵循单一原则，每个 `Loader` 只做一种"转义"工作。 每个 Loader 的拿到的是源文件内容（`source`），可以通过返回值的方式将处理后的内容输出，也可以调用 `this.callback()` 方法，将内容返回给 webpack。 还可以通过 `this.async()` 生成一个 `callback` 函数，再用这个 `callback` 将处理后的内容输出出去。 此外 webpack 还为开发者准备了开发 loader 的工具函数集 —— `loader-utils`。

### 简述如何编写 Plugin

webpack 在运行的生命周期中会广播出许多事件，Plugin 可以监听这些事件，在合适的时机通过 Webpack 提供的 API 改变输出结果。

### DefinePlugin 的妙用

参考 [文章](https://github.com/akira-cn/FE_You_dont_know/issues/14)

下面的代码如果不借助 DefinePlugin 的话会有挺多冗余，而如果使用 DefinePlugin，则可以用来实现类似于宏替换的功能，经过 Terser 等压缩插件的处理，可以减少代码量。

> 注意：DefinePlugin 做的是代码中的宏替换，不要把它当做定义变量来使用。如果在模块中，有与宏名相同的变量，那么这个宏就并不会被替换

```js
function createContext2D() {
  if (
    typeof document !== "undefined" &&
    typeof document.createElement === "function"
  ) {
    // 如果是浏览器环境
    const canvas = document.createElement("canvas");
    return canvas.getContext("2d");
  }
  if (typeof wx !== "undefined" && typeof wx.createCanvas === "function") {
    // 如果是微信小游戏环境
    const canvas = wx.createCanvas();
    return canvas.getContext("2d");
  }
  if (
    typeof wx !== "undefined" &&
    typeof wx.createCanvasContext === "function"
  ) {
    // 如果是微信小程序环境
    return wx.createCanvasContext("canvas");
  }
  return null;
}
```

相关配置：

```js
// webpack.config.js
plugins: [
  new webpack.DefinePlugin({
    'typeof document': env.platform === 'browser' ? '"object"' : '"undefined"',
    'typeof document.createElement': env.platform === 'browser' ? '"function"' : '"undefined"',
    'typeof wx': env.platform !== 'browser' ? '"object"' : '"undefined"',
    'typeof wx.createCanvas': env.platform === 'minigame' ? '"function"' : '"undefined"',
    'typeof wx.createCanvasContext': env.platform === 'miniprogram' ? '"function"' : '"undefined"',
  }),
  // ...
],
// package.json
"scripts": {
  "compile:browser": "webpack --env.platform=browser --env.mode=production",
  "compile:minigame": "webpack --env.platform=minigame --env.mode=production",
  "compile:miniprogram": "webpack --env.platform=miniprogram --env.mode=production",
}
```

## 性能相关

- [webpack-性能优化](http://echizen.github.io/tech/2019/03-19-webpack-performance)

### 如何提高 Webpack 构建速度

- 升级到最新版本的 webpack 和最新版本的 npm
- 配置精确的 resolve.modules 和 extensions，缩小搜索范围，减少文件查找
- `module.noParse` 字段让 Webpack 忽略对部分没采用模块化的文件的递归解析处理，比如`module: {noParse: [/react\.min\.js/]}`（被忽略掉的文件里不应该包含 import 、 require 、 define 等模块化语句，因为 webpack 把它当成一个整体，相当于编译后的文件）
- babel 的处理范围以及缓存（不推荐正式环境使用）
- terser 压缩的多进程以及缓存
- 使用 DLLPlugin 减少公共库模块的编译次数
- 使用 HappyPack 开启多进程处理 Loader 转换
- loader 使用 include 或 exclude，设置合理的处理范围

### 如何利用 Webpack 做前端性能优化

- 路由分割，按需加载
- 使用正确的 hash（contenthash），防止破坏缓存
- 合理地分割公共代码，最大化地利用缓存
- 代码压缩
- 使用 ignorePlugin 减少第三方库无用的代码，比如 moment 的国际化文件，`new webpack.IgnorePlugin(/^\.\/locale$/, /moment$/)`
- 使用 Tree Shaking 减少无用的代码
- 合理使用 DefinePlugin 根据不同平台编译，进行代码量的减少
- 图片压缩
- 多页面的情况下，runtimeChunk 设置为`single`，单独分出一个 bundle
- CSS 代码分离
- Scope Hoisting，符合的 module 会被合并到一个函数中，减少冗余代码，在一定程度上也提升了运行速度（Webpack v4 中已在 production mode 默认）

---

参考：

- [关于 webpack 的面试题总结 - 知乎](https://zhuanlan.zhihu.com/p/44438844)

- [Webpack（三）—hash 和 chunkhash、chunkhash 的使用场景与区别](<https://shengyur.github.io/2018/12/01/Webpack(%E4%B8%89)hash%E5%92%8Cchunkhash%E3%80%81contenthash%20/>)

- [一个被忽视的 webpack 插件](https://github.com/akira-cn/FE_You_dont_know/issues/14)

- [Webpack Hot Module Replacement 的原理解析](https://github.com/Jocs/jocs.github.io/issues/15)
- [webpack-性能优化](http://echizen.github.io/tech/2019/03-19-webpack-performance)
