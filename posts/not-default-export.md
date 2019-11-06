ES Module 中有两种导出方式，默认导出（default export）以及命名导出（named export）。虽然两者可以在一个文件中同时使用，但是建议每次只使用一个。而最开始默认导出是为了更加轻松地与成千上万个导出单个值的 CommonJS 模块进行相互操作。

Node.js 目前试验性的 ESM 支持中，如果一个 ES module 要导入一个 CommonJS 模块，必须只能使用默认导入，因为 _在 nodejs 中 module.exports 总是映射成 ES module 的 default 上导出_。从下面的例子 2 中的原生报错可以看出。

至于 ESM 以及 CommonJS 的相互调用，请见 [cjs 的模块如何和 es6 的模块相互调用？](https://www.zhihu.com/question/288322186)

```js
// 例子 1
// cmj1.js
module.exports = function() {}
// emj1.js
import say from './cmj1.js'
say()

// 例子 2
// cmj2.js
module.exports = {
  say: function(){}
}
// emj2.js
// import { say } from './cmj2.js' // 原生报错，而 Webpack 好像有一层转换不会报错
import cmj from './cmj2.js'
cmj.say()
```

Q: 为什么不推荐使用默认导出？

A: 有很多原因：
- 在源文件重构名称的时候，导入文件的名字不会被同时重构，无法做到一一对应
- 后续想导出更多东西的时候，需要调整导入的语法
- 在重新导出 (re-export) 的时候，`default`必须重命名`export { default as Foo } from "./foo"`；而命名导出不需要`export * from "./foo"`。
- 无法根据 IDE 的智能提示进行判断，需要导入的文件中是否包含默认导出，而命名导出可以且拥有自动补全
- 无法使用自动补全。而命名导出可以在`Foo<tab>`之后自动添加导入`import {Foo} from './foo'`
- 与 CommonJS 同时使用的时候，对转换不友好。比如`const defaultRequire = require('esmodule/foo').default`
- 在与动态导入`import()`使用的时候，转换不友好。（注意`import()`是原生的，而 Webpack 已经帮我们处理了，所以不需要多加一次`default`属性），见下面的例子 1
- 默认导入不支持 Tree-shaking，见下面例子 2

```js
// 例子 1
// 默认导出
const HighChart = await import('https://code.highcharts.com/js/es-modules/masters/highcharts.src.js');
Highcharts.default.chart('container', { ... }); // Notice `.default`
// 命名导出
const { HighChart } = await import('https://code.highcharts.com/js/es-modules/masters/highcharts.src.js');
Highcharts.chart('container', { ... });
```

```js
// 例子 2
// 默认导出 - math.js
function square ( x ) {
	return x * x;
}
function cube ( x ) {
	return x * x * x;
}
export default {
  square,
	cube
}
// 对应的导入 - index.js - 无法 TreeShaking
import math from './math'
console.log(math.cube(5))

// 命名导出
function square ( x ) {
	return x * x;
}
function cube ( x ) {
	return x * x * x;
}
export {
  square,
	cube
}
// 对应的导入 - index.js - 可以 TreeShaking
import { cube } from './math'
console.log(cube(5))
// 或者使用 namespace import 的方式，在 Rollup 中同样可以 TreeShaking，而 Webpack 需要在 V5 版本才支持
import * as math from './math'
console.log(math.cube(5))
```

* [Avoid Export Default · TypeScript Deep Dive](https://basarat.gitbooks.io/typescript/docs/tips/defaultIsBad.html)
