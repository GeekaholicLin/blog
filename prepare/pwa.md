Service worker 是浏览器和网络间的代理。通过拦截文档中发出的请求，service worker 可以直接请求缓存中的数据，达到离线运行的目的。

![](http://image.geekaholic.cn/20191101161328.png@0.8)

```js
// main.js
// 主线程中注册
navigator.serviceWorker.register('/service-worker.js');

// service-worker.js
// service worker 线程中安装、激活、拦截、响应等
// Install （安装）
self.addEventListener('install', function(event) {
    // ...
});
// Activate （激活）
self.addEventListener('activate', function(event) {
    // ...
});
// 监听主文档中的网络请求
self.addEventListener('fetch', function(event) {
    // ...
    // 返回缓存中的数据
    event.respondWith(
        caches.match(event.request);
    );
    // ...
});
```

* [Web Worker、Service Worker 和 Worklet | TinyShare](https://tinyshare.cn/post/HpDVBvTWbUD)
* [PWA 一隅 - 掘金](https://juejin.im/post/5b87c984e51d4559a81ee575)
* [腾讯浏览服务-Service Worker 最佳实践](https://x5.tencent.com/tbs/guide/serviceworker.html)