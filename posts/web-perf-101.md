一个快速的网站包含两部分：加载迅速以及响应迅速。前者包括两方面，一是服务器响应迅速，二是**资源加载迅速以及渲染迅速**，后者一般指页面的流畅比如滚动流畅。

主要讲如何加速资源的加载以及渲染。

### JavaScript

- 极简化（minify），减少加载的字节数
- 脚本资源加载策略的正确选择（async 或 defer），默认情况下`script`标签资源的下载和运行阻塞 HTML 的解析（scripts block parsing）。但是现代浏览器在遇到`script`标签后为了节约时间依然会继续 HTML 的解析，只不过不会渲染，这里且当作会阻塞解析
- 代码分割，按需加载并减少首次加载的字节数
- 减少依赖代码文件（包括但不限于 Tree-Shaking）。比如`moment.js`的国际化文件，`lodash`未使用的方法等

### CSS

- 极简化（minify），减少加载的字节数
- 正确划分关键 CSS（Critical CSS，指初次渲染页面时需要的样式，一般是结构样式）与非关键 CSS，因为 CSS 会阻塞渲染（styles block rendering）。对于关键 CSS 用内联的方式嵌入到 HTML 文件（可以借助工具库`critical`），而非关键 CSS 可以使用`preload`预加载`<link rel="preload" href="other.css" as="style" onload="this.rel = 'stylesheet'" />`。更多可以查看 [Extract Critical CSS](https://web.dev/extract-critical-css/) 以及 [Understanding Critical CSS](https://www.smashingmagazine.com/2015/08/understanding-critical-css/)

### HTTP

- 极简化（minify）HTML 文件
- 对**文本资源**使用 Gzip（可以 Nginx 或 Apache 开启）
- 使用 CDN
- 合理使用 Preloading。比如`<link rel="dns-prefetch">`可以用来预加载 CDN、字体或其他资源的 IP 地址。`<link rel="prefetch">`可以用来在后台低优先级地预加载下一个页面使用的资源。`<link rel="preload">`可以用来在后台高优先级地预加载即将使用的资源，比如非关键 CSS 样式文件

### 图片

- 选择合适的图片格式。比如矢量图使用 svg，尽量使用视频替换 gif 图片
- 图片压缩，可以从合适的大小、质量以及渐进式入手
- 图片懒加载

### 其他

- 相关资源的缓存
- DNS 的预取（dns-prefetch）
