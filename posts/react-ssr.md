## 背景

「服务器端渲染」SSR 不是新奇的概念，在 Web 开发早期很多都是使用的服务器端渲染输出页面（直出页面，比如`php`和`jsp`）。但是因为前后端耦合使得协作效率低下，且大部分是字符串拼接的方式使得维护困难。后来演化成了「前后端分离」，中间的数据传输使用 Ajax，数据的表示一般使用 JSON 格式，。而随着「单页面应用」SPA 框架的出现，大大方便了前端应用的开发（组件化等），但是 SPA 也有缺点。一是首屏加载速度慢，因为要下载框架代码和应用代码，之后框架代码还要执行转换为 HTML，渲染真实 DOM；二是不利于搜索引擎优化 SEO（Seach Engine Optimization），因为 SPA 的渲染是在 JavaScript 执行逻辑中，而大部分搜索引擎只认 HTML，所以难以收录。

所以，SPA 框架提供了在服务器端渲染的能力，将 SPA 与 SSR 结合。这使得开发者不需要手动拼接字符串（框架帮助处理转换），得益于虚拟 DOM，**大部分**代码编写和客户端渲染的时候一样，而这又被称为「同构」（Isomorphic，采用一套代码，整合客户端与服务端逻辑）。这样就解决了单纯使用 SPA 在客户端渲染的两个缺点了：因为已经将 SPA 在服务端编译成 HTML，所以首屏加载速度只需要相关业务代码而不需要 SPA 框架的渲染；再者，编译成 HTML 意味着对搜索引擎友好，能被更好地收录。

## 过程

整个同构的过程为：

node 服务端接收请求 -> 通过请求 path 解析并匹配路由，查找组件 -> 预取相应数据（调用组件规定的方法如 Nextjs 中的`static getInitialProps`），并注入组件 -> 整体渲染成 HTML 字符串（`ReactDOMServer.renderToString`或`ReactDOMServer.renderToNodeStream`）或 stream 流并返回 (-> 「数据注水」，将预取的数据注入到页面方便浏览器访问） -> 网络传输到客户端 (-> 「数据脱水」，将服务器端注入的数据获取，并传入组件 -> ReactDOM.hydrate 对比渲染 -> 完成事件的绑定和处理）

同构中需要让相同的代码在客户端和服务端**各自执行一遍**（但是并不包括路由）

详细的 Demo 可以看 [koa-react-ssr-render](https://github.com/baiyuze/koa-react-ssr-render)

## 缺点

可以看出同构存在一些缺点：

- 相对于纯静态，同构需要处理，占用更多 CPU 和内存
- 开发复杂度有所提高，常用的浏览器 API 无法正常使用（在 Node 的生命周期函数，但是`componentDidMount`依然可以使用浏览器 API），代码中有用到的时候需要对运行环境进行判断或特殊处理。特别是第三方库，在后续引入的时候还要思索支不支持 SSR 以及怎么去支持。
- 涉及服务端，需要学习成本，比如内存监控、性能、缓存策略等。

> 除非项目特别依赖搜索引擎流量，或者对首屏时间有特殊的要求，否则不建议使用 SSR

## 注意事项

同构中一些需要注意的点：

- `renderToString` 和 `renderToStaticMarkup`：前者会为组件增加`checksum`，而 React 在客户端通过 checksum 判断是否需要重新 render。相同则不重新 render，省略创建 DOM 和挂载 DOM 的过程，接着触发 `componentDidMount` 等事件来处理服务端上的未尽事宜 （事件绑定等），从而加快了交互时间；不同时，组件将客户端上被重新挂载 render。后者输出的组件为纯静态 HTML，没有与 React 交互以及额外的 React 属性从而体积更小，但会全部替换掉服务端渲染的内容而造成页面闪烁，不推荐使用。
- 服务端上的数据状态与同步给客户端：服务端上的产生的数据需要随着页面一同返回，客户端使用该数据去 render，从而保持状态一致。服务端上使用 renderToString 而在客户端上依然重新挂载组件，这种情况大多是因为在返回 HTML 的时候没有将服务端上的数据一同返回，或者是返回的数据格式不对导致
- 服务端需提前拉取数据，客户端则在 componentDidMount 调用：可以将预取数据方法放入静态方法中进行复用
- 平台区分：服务端和客户端共用一份代码，有些模块或全局对象无法同样。解决这类问题的方法有：1. 使用两个平台都可以使用的三方库，比如`isomorphic-fetch`; 2. 通过不同端的 webpack 配置`resolve.alias`指定相同名字为不同的模块；3. 使用 webpack 的 DefindePlugin 在不同端定义不同的常量值，根据该值进行不同平台的区分而不会增加文件大小。
- 路由的处理：同构中需要让相同的代码在客户端和服务端**各自执行一遍**，但是并不包括路由的代码，造成这种原因主要是因为客户端是通过地址栏来渲染不同的组件的，而服务端是通过请求路径来进行组件渲染的。服务端路由使用`StaticRouter`而客户端路由使用`BrowserRouter`或`HashRouter`。`StaticRouter`的大概原理是：能够根据传入的 req 对象，在服务器端匹配到将要显示的组件只需要调用 ReactDom 提供的 renderToString 方法，就可以得到 App 组件对应的 HTML 字符串。
- SSR 异步数据获取：在匹配服务端路由后，如果需要拿到多个组件的初始化 state 用来更新 redux 中的 store，可以使用`react-router-config`库，`matchRoutes(routes, req.path)`返回路径匹配的所有组件，然后调用特定的`static`方法，压入`promisesArr`，最后在`Promise.all`的`onResolved`回调中使用`render`并返回。
- 只直出首屏页面可视内容，其他在客户端延迟处理：这是为了减少服务端的负担，也是加快首屏展示时间

其他使用的 tips 和注意事项可以查看 [React 同构直出优化总结](http://www.alloyteam.com/2016/06/react-isomorphic/)。

另外，如果一个页面，在服务端渲染中，数据源比较多的情况下，我们需要等待所有的请求都返回数据才进行 html 拼接并返回，这样我们页面最终渲染的速度就限制在最迟返回数据的请求上了。对于此种情况，可以使用`Bigpipe`进行优化。

## 其他方式

除了「同构」的方式，还可以使用 Prerender 去解决 SPA 带来的 SEO 问题。它的工作原理是区分用户请求和搜索引擎的抓取，在搜索引擎抓取网站收录的时候，服务端使用 headless 浏览器模拟用户访问 SPA，将 SPA 渲染后的 DOM 抓取给搜索引擎。这解决了 SEO 的问题。优点在于简单，不需要大幅度改造现有代码且可利用缓存。缺点在于尚未解决首屏渲染问题。

注意，上述 Prerender 的解决方式与使用插件`prerender-spa-plugin`或的思路不同。后者是在`webpack build`的时候抓取所配置的页面生成对应的静态页面（或页面部分），属于 webpack 编译阶段预生成页面。`prerender-spa-plugin`这种不适用与大量路由页面以及动态内容的情况，而前者属于在运行阶段模拟用户访问生成页面，更加灵活。

## 参考

* [React 中同构（SSR）原理脉络梳理 - 掘金](https://juejin.im/post/5bc7ea48e51d450e46289eab)
* [React 同构直出优化总结 | AlloyTeam](http://www.alloyteam.com/2016/06/react-isomorphic/)
* [从零开始 React 服务器渲染（SSR）同构😏（基于 Koa） - 掘金](https://juejin.im/post/5c627d9b6fb9a049f23d3e38#heading-15)
