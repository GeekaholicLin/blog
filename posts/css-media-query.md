当要根据设备的常规类型（例如打印机或屏幕）或特定特征和参数（例如屏幕分辨率或浏览器视口宽度）来修改网站或应用程序时，媒体查询这时候就派得上用场了。

> 注意：媒体查询是相对于浏览器的，如果媒体查询@media 使用的是相对单位，如 rem，其不是相对于 HTML，而是相对于浏览器默认字体大小（一般是 16px）

## 常见的用法

```html
<!-- 1⃣️ 标签上的用法 -->
<!-- style 标签的 media 属性 -->
<style type="text/css" media="(max-width: 1000px)">
p {
  color: black;
}
</style>
<!-- link 标签 的 media 属性 -->
<!-- ⚠️ 即使媒体查询为 false 也会下载，并在 true 的时候应用样式 -->
<link rel="stylesheet" media="(max-width: 800px)" href="example.css" />
<!-- picture 标签下的 source 的 media 属性 -->
<picture>
    <source srcset="/media/examples/surfer-240-200.jpg"
            media="(min-width: 800px)">
    <img src="/media/examples/painted-hand-298-332.jpg" />
</picture>
<!-- 还有其他支持 media 属性的标签，比如`a`和`area`，比较少见 -->
<map name="planetmap">
  <area shape="rect" coords="0,0,82,126" alt="Sun" href="https://public.xp.cn/down/course_demo/html/area_tabs/sun.htm" media="screen and (min-color-index:256)">
</map>
<!-- 2⃣️ CSS 规则上的用法 -->
<!-- @import 规则 -->
<style>
@import url('landscape.css') screen and (orientation:landscape);
</style>
<!-- @media 规则 -->
<style>
@media (max-width: 600px) {
  #con{
    width: 100px;
  }
}
</style>
<!-- 3⃣️ JS 的用法 -->
<script>
  var mql = window.matchMedia('(max-width: 600px)'); // 根据字符串返回一个`MediaQueryList`对象，如果 window.matchMedia 无法解析 mediaQuery 参数，matches 属性返回的总是 false，而不是报错。
  function screenTest(e) { // `e`为`MediaQueryListEvent`对象，有着两个只读属性值：表示是否匹配的`matches`，以及对应媒体查询的字符串`media`
    if (e.matches) {
      /* 视口宽度小于等于 600px */
      document.body.style.backgroundColor = 'red';
    } else {
      /* 视口宽度大于 600px */
      document.body.style.backgroundColor = 'blue';
    }
  }
  mql.addListener(screenTest); // 注意是`addListener`且没有事件类型，监听媒体查询，用于旧浏览器。**只有在突变的时候触发**，比如 509px 到 601px，或者 601px 到 509px。所以要在未变化的时候显示相应的效果，则需要**提前调用一次**回调。

  // 后续不需要的时候，移除监听器
  mql.removeListener(screenTest)
  // mql.addEventListener('change', screenTest) // 用于新浏览器
  // mql.removeEventListener('change', screenTest)
</script>
```

## 语法

媒体查询由一个可选的「媒体类型」（media type）和任意数量的「媒体特性」（media feature）表达式组成。可以使用「逻辑运算符」以多种方式组合多个媒体查询。媒体查询**不区分**大小写。

### 媒体类型

在新的规范里边，已经有很多的媒体类型已经废弃，常见的媒体类型如下：

- all: 默认值，适配所有设备类型。在很多时候，不需要显式使用，但是如果使用了`not`以及`only`关键字，必须要显式指明为`all`
- print: 主要用于打印预览模式
- screen: 主要用于屏幕，包括电脑屏幕与手机屏幕
- speech: 主要用于屏幕阅读器等发声设备

### 媒体特性

_必须使用括号`()`包裹起来，否则无效。_

常见的媒体特性有：

- aspect-ratio：viewport 宽高比，接受 min/max。宽高比描述了输出设备 viewport 的宽高比（该宽高比会随着 resize 而改变）。该值包含两个以“/”分隔的正整数。代表了水平像素数（第一个值）与垂直像素数（第二个值）的比例。例子：`@media (min-aspect-ratio: 1/1) {}`
- device-aspect-ratio：设备宽高比，接受 min/max。设备宽高比描述了输出设备的宽高比。该值包含两个以“/”分隔的正整数。代表了水平像素数（第一个值）与垂直像素数（第二个值）的比例。例子：`@media (device-aspect-ratio:16/9) {}`
- height: 可视区域的高度，接受 min/max
- width: 可视区域的宽度，接受 min/max。例子：`@media (min-width:800px){}`
- device-height：设备高度，接受 min/max
- device-width：设备宽度，接受 min/max。例子：`@media (min-device-width: 1000px){}`
- orientation：指定了设备处于横屏（宽度大于宽度）模式`landscape`还是竖屏（高度大于宽度）模式`portrait`。需要注意的是，在移动端软键盘唤起的时候，`portrait`可能会变成`landscape`。例子：`@media (orientation: portrait){}`

- device-pixel-ratio：设备像素比，接受 min/max。`@media only screen and (-webkit-min-device-pixel-ratio: 2), only screen and (min-device-pixel-ratio: 2)`。但这是非标准媒体特性，经常需要和标准的媒体特性`resolution`进行搭配。
- resolution: 指定输出设备的**像素密度**，接受 min/max。接受`dpi`以及`dppx`作为值，其中`dppx`的兼容性较差，`1 ddpx`等于`1 dpr`

```scss
// 代码来自：https://css-tricks.com/snippets/css/retina-display-media-query/
// 兼容比较高的设备
@media
(-webkit-min-device-pixel-ratio: 2),
(min-resolution: 192dpi) {
    /* Retina-specific stuff here */
}
// 兼容比较老的设备
@media
only screen and (-webkit-min-device-pixel-ratio: 2),
only screen and (   min--moz-device-pixel-ratio: 2), /* Looks like a bug, so may want to add: */
only screen and (   -moz-min-device-pixel-ratio: 2),
only screen and (     -o-min-device-pixel-ratio: 2/1),
only screen and (        min-device-pixel-ratio: 2),
only screen and (                min-resolution: 192dpi),
only screen and (                min-resolution: 2dppx) {
    /* Your retina specific stuff here */
}

```

### 逻辑运算符

- and: 表示「且」
- 逗号分隔符（`,`）：表示「或」
- not：表示「非」，使用的时候必须指定一个媒体类型。如果中间有逗号分隔符（有很多个媒体查询），`not`只会用于所在的媒体查询，比如`@media not screen and (color), print and (color)`相当于`@media (not (screen and (color))), print and (color)`
- only：表示「有且仅有」，使用的时候必须指定一个媒体类型。主要用于防止不支持「媒体特性」的旧浏览器应用了样式。比如`screen and (max-width: 500px)`，在不支持媒体特性的浏览器中只会识别`screen`，从而使用了对应的样式。如果使用了`only screen and (max-width: 500px)`，会先读取`only`而不会有一个叫做`only`的设备，所以不会应用样式。所以，在需要兼容不支持「媒体特性」的旧浏览器而又有些媒体查询使用了「媒体特性」，那使用了「媒体特性」的媒体查询最好加上`only`关键字（如果只兼容支持「媒体特性」的，可以不加）

### 例子

这里给出常见的例子，增强一下印象。

```scss

@media (max-width: 600px) {}
@media (min-device-width: 1000px){}
@media screen and (min-width:700px) and (orientation:landscape){}
@media screen and (device-aspect-ratio: 16/9), screen and (device-aspect-ratio: 16/10) { }
@media (orientation: portrait){}
@media (min-resolution: 90dpi){}

```

也可以借助 scss 的高级特性，进行一定的封装，简化写法且将其放入选择器中。这种的好处是媒体查询跟着选择器走，就跟组件一样。而坏处就是编译后会存在冗余。从 [Sass Guidelines](https://sass-guidelin.es/zh/#section-41) 中获取更多详情。

```scss
$device-ratio: 1 2 3 4;
$str: min-device-pixel-ratio;
@mixin device($specific: $device-ratio, $query: $str){
  @each $ratio in $specific {
    @media (-webkit-$query: $ratio),($query: $ratio){
     @content
    }
 }
}

body, html {
  color: black;
  @include device{
    background-color: red;
  }
}
```

---

参考：

* [深入理解 CSS Media 媒体查询 - 小火柴的蓝色理想 - 博客园](https://www.cnblogs.com/xiaohuochai/p/5848612.html)
* [CSS 媒体查询 - 掘金](https://juejin.im/post/5affd7ff6fb9a07aa2139ebb)
* [Using media queries - CSS: Cascading Style Sheets | MDN](https://developer.mozilla.org/en-US/docs/Web/CSS/Media_Queries/Using_media_queries)