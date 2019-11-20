# 等待去探索

## Koa

中间件的机制是洋葱模型（先从外到里，再从里到外），和`Connect`模块的流水模型（或者说管道模型）不大相同

属性：

- proxy
- subdomainOffset
- env
- request

Koa 中的错误处理相关：[Error-Handling](https://github.com/koajs/koa/wiki/Error-Handling) 以及 [Koa 中的错误处理](https://www.cnblogs.com/Wayou/p/error_handling_in_koajs.html)

* [koa-middlewares - npm](https://www.npmjs.com/package/koa-middlewares)
* [cnpm/koa-middlewares: easy way to use some useful koa middlewares](https://github.com/cnpm/koa-middlewares)
* [favicon/index.js at master · koajs/favicon](https://github.com/koajs/favicon/blob/master/index.js)
* [Node.js 安全清单 - 后端 - 掘金](https://juejin.im/entry/5a9767def265da4e896b05b4)
* [venables/koa-helmet: Important security headers for koa](https://github.com/venables/koa-helmet)
* [koajs/static: Static file server middleware](https://github.com/koajs/static)
* [queckezz/koa-views: Template rendering middleware for koa (hbs, swig, pug, anything! )](https://github.com/queckezz/koa-views)
* [Koa2 中间件原理解析 —— 看了就会写 - 掘金](https://juejin.im/post/5ba7868e6fb9a05cdf309292#heading-8)
* [koa2 一网打尽（基本使用，洋葱圈，中间件机制和模拟，源码分析（工程，核心模块，特殊处理），核心点，生态） - - SegmentFault 思否](https://segmentfault.com/a/1190000017370447#articleHeader17)
* [KOA2 框架原码解析和实现 - 掘金](https://juejin.im/post/5c8f6f53e51d4516f6680012)
* [chenshenhai/rollupjs-note: 《Rollup.js 实战学习笔记》已完结 😆](https://github.com/chenshenhai/rollupjs-note)
* [回炉重学 HTML/DOM/Element/Node 之间的关系 · Issue #34 · chenshenhai/blog](https://github.com/chenshenhai/blog/issues/34)
* [koa2 进阶学习笔记 · GitBook](https://chenshenhai.github.io/koa2-note/)
* [Koa.js 设计模式-学习笔记 · GitBook](https://chenshenhai.github.io/koajs-design-note/)
* [深入 koa2 源码 - 知乎](https://zhuanlan.zhihu.com/p/54714710)

## Nodejs

Node 的模块加载机制

* [[源码解读] 一文彻底搞懂 Events 模块 - 掘金](https://juejin.im/post/5d69eef7f265da03f12e70a5)
* [【译】你所要知道关于 Node.js Streams 的一切 - 知乎](https://zhuanlan.zhihu.com/p/44809689)
* [i5ting/How-to-learn-node-correctly: [全文] 如何正确的学习 Node.js](https://github.com/i5ting/How-to-learn-node-correctly)
* [i5ting_ztree_toc:README](https://i5ting.github.io/How-to-learn-node-correctly/#1090602)
* [七天学会 NodeJS](http://nqdeng.github.io/7-days-nodejs/)
* [Node.js 之 log4js 完全讲解 - 知乎](https://zhuanlan.zhihu.com/p/22110802?refer=FrontendMagazine)
* [chyingp/nodejs-learning-guide: Nodejs 学习笔记以及经验总结，公众号"程序猿小卡"](https://github.com/chyingp/nodejs-learning-guide)
* [Introduction · nodebook](https://yunnysunny.gitbooks.io/nodebook/content/)
* [alsotang/node-lessons: 《Node.js 包教不包会》 by alsotang](https://github.com/alsotang/node-lessons)
* [require() 源码解读 - 阮一峰的网络日志](http://www.ruanyifeng.com/blog/2015/05/require.html)

## TypeScript

* [Introduction · TypeScript Deep Dive](https://basarat.gitbooks.io/typescript/)

## 可视化

* [图分析与图可视化：挑战与机遇 - 掘金](https://juejin.im/post/5daeea91518825636658298e)

### D3js

### Canvas

### SVG

## Web Component

## Webassembly

## 函数式

### Rxjs

### Ramdajs

### ReasonML

## Nginx

* [nginx 反向代理跨域基本配置与常见误区 - 木子墨 - 博客园](https://www.cnblogs.com/heioray/p/9529566.html)

## Docker

## Web Performance

* [前端性能优化之浏览器渲染优化 —— 打造 60FPS 页面 · Issue #9 · fi3ework/blog](https://github.com/fi3ework/blog/issues/9)

## Network

## Web Security

## Vue

### 前言

#### Vue 三要素

- 响应式：如何监听数据变化
- 模版引擎：如何解析模版
- 渲染：如何将监听到的变化和解析后的 HTML 进行渲染

#### 双向绑定实现方式

- KnockoutJS 的观察者模式
- Angular 的脏检查
- Ember 基于数据模型的双向绑定
- Vue 使用的`Object.defineProperty`与`Proxy`

### 数据劫持的优势

- 无需显示调用：比如`data.name='GeekaholicLin'`即可触发，不像 React 需要使用`setState`

- 可精确得知变化数据：劫持了属性的 setter，当属性值改变，我们可以精确获知变化的内容

* [面试官：实现双向绑定 Proxy 比 defineproperty 优劣如何？- 掘金](https://juejin.im/post/5acd0c8a6fb9a028da7cdfaf)
* [剖析 Vue 原理&实现双向绑定 MVVM - 前端足迹 - SegmentFault 思否](https://segmentfault.com/a/1190000006599500)
