### 简介

Babel 是 JavaScript 编译器（或者说转换器），用于处理 JavaScript 的兼容问题。

### 使用方法

- 单个文件：文件顶部添加`require(babel-register)({ presets: ['latest'] })`
- 命令行：安装`babel-cli`，使用命令行`babel src -d lib -s`
- 浏览器使用：通过`script`标签引入，`<script src="https://unpkg.com/@babel/standalone/babel.min.js"></script>`
- 作为 Webpack 等构建工具的插件`babel-loader`等
- 直接使用`babel-core`，用编程的方式使用

使用方式有所不同，但实际上都是调用的`babel-core`。`babel-core`的作用在于：

- 加载和处理配置
- 加载插件
- 调用`babylon`解析器将代码处理成 AST
- 调用`babel-traverse`遍历 AST，并应用插件对 AST 进行转换
- 调用`babel-generator`生成代码，包括 source map 以及源代码

### 运行方式

Babel 总共分为三个阶段：解析（parse）、转换（transform）以及生成（generate）。

![Babel 运行过程](http://image.geekaholic.cn/20191107114744.png@0.8)

- 解析：将代码通过 Babel 编译器`babylon`（基于 [acorn](https://github.com/acornjs/acorn)，后改名为`@babel/parser`）转换为 [ESTree](https://github.com/estree/estree) 规范（此规范为 SpiderMonkey 引擎输出的 JavaScript AST 规范） 的 AST 抽象语法树
- 转换：通过`babel-traverse`遍历访问 AST 节点，并对其进行相关操作（`babel-core`中的相关`transform`接口），进行转换生成新的 AST。这是最复杂的部分，也是 Babel 插件介入的部分。
- 生成：使用`babel-generator`以新的 AST 生成新的代码，同时还可以创建 source map

Babel 本身不具有转换功能，转换功能都被分解到 plugin 中，所以当未配置插件的时候，经过 Babel 后输出的代码与输入代码是相同的。

Babel 只是转译新标准引入的**语法**，比如 ES6 箭头函数：而新标准引入的新的原生对象，部分原生对象新增的原型方法，新增的 API 等（Proxy、Set 等）, 需要引入对应的 polyfill 来解决

preset 是 plugin 的集合，所有的 plugin 会运行在所有的 presets 之前，plugins 的插件顺序是顺序的从前到后，而 preset 是逆序的从后到前（这是因为大多数用户的错误使用而使得 Babel 的向后兼容）。

plugin 的种类分为转换插件以及语法插件。转换插件用于转换代码，将源码进一步转换并且输出，这也是最常用的 Babel 功能。而语法插件允许 Babel 解析特定类型的语法（比如`jsx`、`flow`以及），而如果已经使用了相应的转换插件，则可以不指定语法插件，比如说已经使用`@babel/preset-react`就可以不需要指定`jsx`。

前面讲到，presets 其实是一组插件的集合。比如`stage-0,1,2,3`、`env`、`react`、`flow`、`typescript`、`minify`等。其中`stage-0,1,2,3`分别对应了规范的阶段，`env`的前身是已经废弃的`latest`，包含了所有的`es201x`。且随着更加灵活的`env`出现，`es201x`也被废弃。

`env`的核心在于可以通过配置（`browserlist` 配置语法）获得目标环境`target`，然后读取 [`compat-table`](https://github.com/kangax/compat-table) 对目标环境已有的功能不做转换，只对未有的功能做必要的转换，减少代码量。（在引入`core-js@3`后，对于`core-js@2`版本依然使用`compat-table`，而`core-js@3`版本使用`core-js`官方维护的`core-js-compat`数据）

plugins 和 presets 的添加用法可以有具体路径，`node_modules`目录下长名称和短名称。且两者都支持参数的配置，_插件名在前参数在后组成配置数组_。

```js
{
  "plugins": [
    "./node_modules/asdf/plugin",
    "myPlugin", // node_modules 目录下查找
    "babel-plugin-myPlugin", // 短名称写法
    "@org/babel-plugin-name",
    "@org/name", // scope 类型也支持短名称
    [
      "env",
      {
        "loose": true,
        "modules": false
      }
    ]
  ]
}
```

### 配置方式

- 单个文件的形式使用`babel-register`的 options 传入
- 命令行使用参数`--presets`等传入
- 构建工具的插件使用对应的`options`传入
- `package.json`中的 babel 字段
- 项目根目录中的`.babelrc`、`.babelrc.js`配置文件或 babel v7 中新增的`babel.config.js`配置文件

### Babel 中的配套工具

除了上述讨论的配置方法，Babel 还提供了需要配套工具`babel-*`。

- babel-cli：命令行工具，用来编译文件，常用于尚未使用 webpack 等构建工具的小项目
- babel-node: Babel 7 之前是 babel-cli 的一部分，不需要单独安装。用来在 Node 环境中执行 ES6+的代码而不需要额外进行编码，比如`babel-node es2015.js`。相当于`babel-node = babel-polyfill + babel-register`
- babel-register: 在文件顶部使用，给当前文件加上钩子。在当前文件被`require`的时候，先使用 babel 进行转换。因为是实时转码，**只适合在开发环境使用**。注意`babel-register`并没有引入`polyfill`，需要手动引入。
- babel-polyfill: Babel 本身只做语法转换，不对 API（比如`Map`、`Set`、`Generator`等实现及其方法，`Array.from`等新方法）处理，而`babel-polyfill`则是增加相关 API 的 polyfill，用现有语法去支持新的 API。其有两个缺点，一是打包体积大因为加载了全部的 polyfill（只用到`Array`却加载全部的`Map`、`Set`等），**需要一定的配置**才能做到按需加载；二是修改了原型链，污染全局变量。
- babel-runtime: 作为`dependency`使用，内部集成了`core-js`帮助转换内置类比如`Map`等及其**静态方法**；集成了`regenerator`对`generator/yield`以及`async/await`的支持，是对`core-js`的有效补充；还集成了`helper`函数，比如`asyncToGenrator`等，与`babel-plugin-transform-runtime`的搭配使用可以避免 babel 编译的工具函数在每个模块里重复出现。缺点是不能模拟**实例方法**，比如`Array.prototype.flat`
- babel-plugin-transform-runtime: 作为`devDependency`使用，必须将`babel-runtime`作为依赖。做了三件事：一是在使用 generator/async 的时候自动引入`@babel/runtime/regenerator`；二是通过引用`babel-runtime`中相关的`helper`函数达到复用的目的，在编译打包的时候只打包一份，减少代码量。三是通过引用`core-js`这一标准 polyfills 库进行无缝地 polyfill，让用户无感知地使用内建 API，并且可以按需引入以及不会全局污染。
- babel-loader: 集成到 webpack 中从而在 webpack 编译周期内进行代码转换，方便后续压缩混淆等 webpack 的处理

```js
// index.js
require('babel-register')({
  presets: [
    [
      'env',
      {
        modules: 'commonjs',
      }
    ],
    'stage-0',
    'react'
  ]
})
require('babel-polyfill') // <= 需要手动引入
require('./server/main')
```

### Babel 7 的变更

- 废弃 es201x 、latest 以及 stage-x 的 presets，推荐使用 env 以及显示地声明对应功能的 stage-x 插件
- 名称的变化，把`babel-*`重命名为`@babel/*`，比如`@babel/cli`，`@babel/preset-env`（短名称为`@babel/env`）；把`babylon`命名为`@babel/parser`（强依赖于`acorn`与`acorn-jsx`)；而为了区分规范以及提案的插件，`stage-x`中的插件名称中增加`proposal`，比如`@babel/plugin-proposal-function-bind`（`obj::func`转换为`func.bind(obj)`），如果插件名称中包含`es201x`则进行移除，比如`@babel/plugin-transform-classes`，原来是`babel/plugin-transform-es2015-classes`；
- 不支持低版本的 node，要求 nodejs>=6
- `only`以及`ignore`匹配规则中**通配符**向 glob 靠拢，之前的`*.foo.js`包括当前目录下任意目录的文件，与`./**/*.foo.js`效果类似，而目前`*.foo.js`只匹配当前目录
- `@bable/node` 从 `@babel/cli` 中独立，需要的时候要单独安装
- 配置文件及其查找方法的变化
- `babel-runtime`被拆分为两类包，一类是仅包含`helpers`函数以及`regenerator`的`@babel/runtime`，另外一类是类似`babel-runtime`原有功能（`@babel/runtime`+对应的`core-js`）的`@babel/runtime-corejs2`或`@babel/runtime-corejs3`，所以如果需要 polyfill，则安装`@babel/runtime-corejs2`或`@babel/runtime-corejs3`，并通过`@babel/transform-runtime`的`core-js`选项指定是从版本 2 还是版本 3 导入。`@babel/runtime-corejs3`与配置`corejs: 3`一起使用是支持**实例方法**的，而不仅仅是**静态方法**，且通过配置`{ corejs: 3, proposals: true}`可以支持导入提案中的 API 对应的 polyfill！（`@babel/runtime-corejs3`相当于`@babel/runtime`+`core-js@3`，`@babel/runtime-corejs3`同理）

更多变更内容请见 [官方文档](https://babeljs.io/docs/en/v7-migration)。可以使用`babel-upgrage`工具进行自动升级，升级内容以及用法请见工具文档。

### Babel 7 配套工具的一些配置说明

此部分内容大部分来自官网的`@babel/env`、`@babel/polyfill`以及`@babel/transform-runtime`的用法。

`core-js`是 polyfill 提供者（`@babel/polyfill`与`@babel/runtime-corejs3`中包含了`core-js`），而`@babel/env`与`@babel/transform-runtime`是 polyfill 注入者。

目前有三种方式在源文件中引入`core-js`的 polyfills：

- `@babel/preset-env`以及`useBuiltIns: 'entry'`，用于引入目标平台不支持的 polyfill
- `@babel/preset-env`以及`useBuiltIns: 'usage'`，用于引入目标平台不支持且在文件中使用的 polyfill
- `@babel/transform-runtime`，用于引入`core-js`的 **ponyfill**（不会污染全局且纯粹的 polyfill），常用于开发包或者库。缺点在于不考虑平台环境，而是按需引入所有`core-js`中已经支持的实现。

#### @babel/polyfill

前面说到`@babel/polyfill`会加载新增 API 的 polyfill，但是会污染全局，且默认情况下会导入全部。为了减少体积，一种方法是应该与`@babel/preset-env`及其`useBuiltIns`选项一起使用，另外一种方法是手动导入所需要的 polyfill 而不是整个包。

这里介绍第一种减少导入的 polyfill 体积的配置方法。

首先，需要在入口文件手动引入`@blabel/polyfill`：

```js
// 方式 1 - 在入口（entry point）文件顶部使用 ES5 语法或 ES6 语法
require('@babel/polyfill')
import('@babel/polyfill')

// 方式 2 - webpack 配置，仅适用于 @babel/polyfill，不适用于 Babel v7.4 之后的 core-js/stable
module.exports = {
  entry: ["@babel/polyfill", "./app/js"],
};
// 方式 3 - 使用 @babel/env 的 userBuiltIns 选项（可按需，推荐）

```

为了减少体积，推荐使用方式三，配置`@babel/env`的`userBuiltIns`选项。userBuiltIns 选项表示的是如何处理 polyfills，可选值为`false`、`'usage'`以及`'entry'`

方式 3 根据`userBuiltIns`的不同取值，需要不同的配置：

- `userBuiltIns` 为 `false`，表示不使用 polyfill -- 不自动转换`import 'core-js'`为相应的 polyfill，也就是会变成**导入全部**的 polyfill
- `userBuiltIns` 为 `'entry'`，需要像方式 1 或方式 2 一样导入到入口源文件顶部（**也就是一个入口只能一次导入**）。这时候会检测使用了`import('@babel/polyfill')`的文件以及相应的`target`，在这些文件上加载不支持 API 的 polyfill
- `userBuiltIns` 为 `'usage'`，**不需要做任何 import 处理**。这时候会自动检测并按需加载，源文件中使用到的相关 API 如果`target`不支持，则在该文件顶部添加对应的 polyfill。

`@babel/polyfill`其实内部已经包含了`core-js`以及`regenerator-runtime`，所以可以不用安装其他的包，部分转换后如下所示，可以看出`core-js`在其内部`lib`文件夹。

```js
// polyfill 转换后
import "@babel/polyfill/lib/core-js/modules/es7.array.includes";
import "@babel/polyfill/lib/core-js/modules/es6.string.includes"
```

从 Babel v7.4 开始，`@babel/polyfill`不再推荐使用，而是推荐其拆分形式。也就是方式 1 中，在相关转换源文件，把`import('@babel/polyfill')`改为以下代码，而方式 2 不能这样更换。

```js
// 需要单独安装特定版本的`core-js`以及`regenerator-runtime`（或者`@babel/runtime`）
// npm i --save core-js regenerator-runtime
// npm i --save core-js @babel/runtime
import "core-js/stable";
import "regenerator-runtime/runtime";
```

拆分的原因是`core-js@3`发布之后，`@babel/polyfill`不能平滑地从`core-js@2`迁移到`core-js@3`。

> This package doesn't make it possible to provide a smooth migration path from core-js@2 to core-js@3: for this reason, it was decided to deprecate @babel/polyfill in favor of separate inclusion of required parts of core-js and regenerator-runtime

#### 实践方案

在 Babel v7.4 之前因为没有出现`core-js@3`：项目开发使用`@babel/polyfill` + `@babel/env`中的`useBuiltIns`选项并取值`entry`或`usage` + `@babel/env`中`corejs: 2` + 每个 entry 入口文件的 import; 前端类库开发使用`@babel/transform-runtime` + `@babel/runtime-corejs2`（但是没有实例方法）

在 Babel v7.4 之后，项目开发使用`core-js@3` + `@babel/runtime` + `@babel/env`中的`useBuiltIns`选项并取值`entry`或`usage` + `@babel/env`中`corejs: 3` + 每个 entry 入口文件的 import; 前端类库开发使用`@babel/transform-runtime`+`@babel/runtime-corejs3`（包含实例方法）

那有没有在`@babel/env`兼顾前端类库开发呢？目前我们可以知道，两者分离是因为，`@babel/env`的 polyfill 会污染全局，`@babel/transform-runtime`不会污染全局但是无法利用 target 的优势。而 Babel 官方在最新的 [RFC](https://github.com/babel/babel/issues/10008) 中讨论，让`@babel/env`支持`useBuiltIns: 'pure'`的配置以开启非全局污染。或者将`target`配置提升到顶级配置项，让`@babel/transform-runtime`读取从而利用 target 的优势。

### Babel 7 配置方式读取方案

Babel v7 的配置文件分为两种，分为项目范围的配置文件（Project-wide）以及相对于文件的配置文件（File-relative）。项目范围的配置文件是`babel.config.js`，而相对于文件的配置文件则是`.babelrc`、`.babelrc.js`以及`package.json`中的 babel 字段。两者可以一起使用也可以单独使用。

Project-wide 配置通常用于 `monorepo` 结构的项目，也就是下边子目录有多个`package.json`，为了共用一套配置文件，可以在项目根目录使用`babel.config.js`。Project-wide 是通过编程选项`configFile`来指定特定文件，`configFile`的默认值为`path.resolve(opt.root, 'babel.config.js')`，`false`的时候为关闭。也就是查找是根据`root`层级查找，而`root`是你当前使用的 babel 命令所在的目录。如果在项目子目录中运行 babel 命令，需要指定`rootMode`为向上查找（`upward`），对找到的`babel.config.js`所在层级设置为`root`，例子如下：

```shell
cd packages/some-package
babel --root-mode upward src -d lib
```

File-relative 的意思是指相对于编译文件开始，向上查找到 package.json 所在目录，且这目录需要在`babelrcRoots`中，通常用于单独的项目。

而`babelrcRoots`配置是做什么用的呢？在默认情况下，Babel 只会在`root`（注意这是指执行 babel 时所在目录）下查找`.babelrc`并应用，因为 Babel 不知道你在子目录中给出的`.babelrc`是否真的打算要加载或者子目录下的插件是否安装好了（因为`.babelrc`可能在`node_modules`文件夹内部或者是软连接的，这些时候插件和预设一般是没有安装好的），所以需要使用`babelrcRoots`来告诉 Babel，编译文件时对拥有`.babelrc`的文件夹下的文件使用`.babelrc`。

```plain
package.json
babel.config.js
.babelrc
packages/
  mod/
    package.json
    .babelrc
    index.js
```

上面的项目结构中，假设在项目根目录运行 Babel 命令，则`root`为项目根目录，`mod/index.js`的配置文件读取的是`babel.config.js`，因为默认不使用`.babelrc`。但是如果使用`babelrcRoots: [".", "packages/*"]`，则告诉 Babel，`packages`下的子目录**内部**如果遇到`.babelrc`，那么编译这些匹配到的目录的文件时候**合并**子目录的`.babelrc`与根目录的`babel.config.js`。对于根目录而言，在上述`babelrcRoots`配置中，会**合并**项目根目录的`babel.config.js`与根目录的`.babelrc`。

上述这种差异化其实**也可以**通过`overrides`配置项实现，通过匹配并应用不同的配置实现差异化。

```js
// 来自 babel 官方仓库
{
  overrides: [
      {
        test: "packages/babel-parser",
        plugins: [
          "babel-plugin-transform-charcodes",
          ["@babel/transform-for-of", { assumeArray: true }],
        ],
      },
      //...
      {
        // The runtime transform shouldn't process its own runtime or core-js.
        exclude: [
          "packages/babel-runtime",
          /[\\/]node_modules[\\/](?:@babel\/runtime|babel-runtime|core-js)[\\/]/,
        ],
        plugins: [
          includeRuntime
            ? ["@babel/transform-runtime", { version: "7.4.4" }]
            : null,
        ].filter(Boolean),
      },
    ]
}
```

所以个人认为最佳实践应该是「项目范围的`babel.config.js`共享公共配置 + 多项目的差异化」，而差异化实现可以是「`babelrcRoots`+多项目下的`.babelrc`」或者是`overrides`

### core-js 与 Babel

以下都是以`core-js@3`为例。

`core-js`分为 3 个包，`core-js`定义全局的 polyfill，会污染全局；`core-js-pure`提供 polyfill 但是不污染全局；`core-js-bundled`已经打包了的全局 polyfill。是的，`@babel/transform-runtime`使用的就是`core-js-pure`，而`@babel/env`中使用的是会污染全局的`core-js`，所以，这也就为什么说，开发前端库的时候推荐使用`@babel/transform-runtime`

`core-js`有着不同的导入形式：

```js
// polyfill all `core-js` features:
import "core-js";
// polyfill only stable `core-js` features - ES and web standards:
import "core-js/stable";
// polyfill only stable ES features:
import "core-js/es";

// if you want to polyfill `Set`:
// all `Set`-related features, with ES proposals:
import "core-js/features/set";
// stable required for `Set` ES features and features from web standards
// (DOM collections iterator in this case):
import "core-js/stable/set";
// only stable ES features required for `Set`:
import "core-js/es/set";
// the same without global namespace pollution:
import Set from "core-js-pure/features/set";
import Set from "core-js-pure/stable/set";
import Set from "core-js-pure/es/set";

// if you want to polyfill just required methods:
import "core-js/features/set/intersection";
import "core-js/stable/queue-microtask";
import "core-js/es/array/from";

// polyfill reflect metadata proposal:
import "core-js/proposals/reflect-metadata";
// polyfill all stage 2+ proposals:
import "core-js/stage/2";
```

但`core-js`一般是与 Babel 一起搭配使用，让 Babel 帮忙进行有效地进行导入，减少体积。而单独使用的时候，为了能够根据平台的支持程度进行导入，官方也提供了`core-js-builder`工具。兼容性数据在`core-js@3`的时候因为`compat-table`的一些缺陷，`core-js`官方维护了`core-js-compat`包，这也是`@babel/env`在使用`core-js@3`的时候用的兼容性数据。

---

参考：

* [一口（很长的）气了解 babel](https://zhuanlan.zhihu.com/p/43249121)
* [一文读懂 babel7 的配置文件加载逻辑](https://segmentfault.com/a/1190000018358854)
* [RFC: Rethink polyfilling story](https://github.com/babel/babel/issues/10008)
* [core-js@3, babel and a look into the future](https://github.com/zloirock/core-js/blob/master/docs/2019-03-19-core-js-3-babel-and-a-look-into-the-future.md)
* [Config Files](https://babeljs.io/docs/en/config-files)
* [@babel/polyfill](https://babeljs.io/docs/en/babel-polyfill)
* [@babel/plugin-transform-runtime](https://babeljs.io/docs/en/babel-plugin-transform-runtime)