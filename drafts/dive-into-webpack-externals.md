## webpack 配置中的 externals

### 用法简介

提供了「从输出的 bundle 中排除依赖」的方法。这是已经知道所创建的 bundle（经常是插件或增强型代码） 中，指定的 externals 依赖，已经存在于开发的应用中，但是又不希望影响在插件逻辑代码中对该依赖的使用，比如`import`或`require`。这样的做法是将 externals 的依赖安装控制器交给插件的使用者，也就是应用开发者必须手动安装确保 externals 存在，而不是插件开发者去确保安装。

它可以防止将某些 import 的包 (package) 打包到 bundle 中，而是在运行时 (runtime) 再去从外部获取这些扩展依赖。

externals 的使用和 npm 中的`peerDependency`十分相似。

常见场景：

- 插件开发：开发过程中依赖主库，但不希望插件的打包将主库打包，因为已经知道使用这个插件的时候，主库一定是存在的。

### 可以使用的模块格式

- root：可以通过一个全局变量访问 library（例如，通过 script 标签）。
- commonjs：可以将 library 作为一个 CommonJS 模块访问。
- commonjs2：和上面的类似，但导出的是 module.exports.default.
- amd：类似于 commonjs，但使用 AMD 模块系统。

其实不止这些，见下边的源码。

### externals 与 libraryTarget 的关系

- libraryTarget 配置如何暴露 library。如果不设置 library, 那这个 library 就不暴露。就相当于一个自执行函数
- externals 是决定的是在引入的时候，以哪种模式去加载所引入的额外的包。而这引入的模式取决于引入包的`libraryTarget`是怎么暴露的。也就是说，externals 的引入模式需要和`libraryTarget`配置的暴露模式保持一致。

### 怎么使用

从 [ExternalModuleFactoryPlugin](https://github.com/webpack/webpack/blob/eeafeee32ad5a1469e39ce66df671e3710332608/lib/ExternalModuleFactoryPlugin.js) 源代码中的`handleExternals`函数可以看出，externals 本身最外层可以是字符串，数组，正则表达式，函数或对象，常用的是对象形式。

#### 字符串

如果`externals`本身的配置是一个字符串，或数组中的元素是字符串，或对象中的键值对的值是字符串都属于此种情况。

字符串中可以包含所指定的引入格式，比如`'commonjs lodash'`。那单纯的字符串`'lodash'`要怎么知道以什么模式引入的呢？答案在 [WebpackOptionsApply](https://github.com/webpack/webpack/blob/74fdbd44fca122d51944b03157ff0ed5c1111c27/lib/WebpackOptionsApply.js) 源代码中。除了`electron-*`的平台相关，需要初始化`globalType`为特定的`commonjs`外，在存在`options.externals`的情况，与`output.libraryTarget`等同，而`libraryTarget`的默认值为`var`，也就是赋值给一个变量。

题外话：在这里也能了解到`target`配置有什么用了吧？用来初始化平台相关的一些特定配置

```js
// 不断向上查找调用的地方，找到 WebpackOptionsApply 文件
// lib/WebpackOptionsApply.js#L227-L233
if (options.externals) {
  const ExternalsPlugin = require("./ExternalsPlugin");
  new ExternalsPlugin(options.output.libraryTarget, options.externals).apply(
    compiler
  );
}
// lib/ExternalModuleFactoryPlugin.js
// 如果没有指定类型，则从配置的字符串中分割，且默认值为 globalType，在这里为 options.output.libraryTarget（默认值为'var'）
const UNSPECIFIED_EXTERNAL_TYPE_REGEXP = /^[a-z0-9]+ /;
if (
  type === undefined &&
  UNSPECIFIED_EXTERNAL_TYPE_REGEXP.test(externalConfig)
) {
  const idx = externalConfig.indexOf(" ");
  type = externalConfig.substr(0, idx);
  externalConfig = externalConfig.substr(idx + 1);
}
callback(
  null,
  new ExternalModule(externalConfig, type || globalType, dependency.request)
);
// lib/ExternalModule.js
// 默认值'var'，对应什么样的模块模版呢？getSourceForDefaultCase 给出了答案，直接返回 variableName
const getSourceForGlobalVariableExternal = (variableName, type) => {
  if (!Array.isArray(variableName)) {
    // make it an array as the look up works the same basically
    variableName = [variableName];
  }

  // needed for e.g. window["some"]["thing"]
  const objectLookup = variableName.map(r => `[${JSON.stringify(r)}]`).join("");
  return `(function() { module.exports = ${type}${objectLookup}; }());`;
};
const getSourceForCommonJsExternal = moduleAndSpecifiers => {
  if (!Array.isArray(moduleAndSpecifiers)) {
    return `module.exports = require(${JSON.stringify(moduleAndSpecifiers)});`;
  }
  const moduleName = moduleAndSpecifiers[0];
  const objectLookup = moduleAndSpecifiers
    .slice(1)
    .map(r => `[${JSON.stringify(r)}]`)
    .join("");
  return `module.exports = require(${JSON.stringify(
    moduleName
  )})${objectLookup};`;
};
const getSourceForDefaultCase = (optional, request, runtimeTemplate) => {
  if (!Array.isArray(request)) {
    // make it an array as the look up works the same basically
    request = [request];
  }
  const variableName = request[0];
  const missingModuleError = optional
    ? checkExternalVariable(variableName, request.join("."), runtimeTemplate)
    : "";
  const objectLookup = request
    .slice(1)
    .map(r => `[${JSON.stringify(r)}]`)
    .join("");
  return `${missingModuleError}module.exports = ${variableName}${objectLookup};`; // 就是返回这一句
};
class ExternalModule extends Module {
  getSourceString(runtimeTemplate, moduleGraph, chunkGraph) {
    const request =
      typeof this.request === "object" && !Array.isArray(this.request)
        ? this.request[this.externalType]
        : this.request;
    switch (this.externalType) {
      case "this":
      case "window":
      case "self":
        return getSourceForGlobalVariableExternal(request, this.externalType);
      case "global":
        return getSourceForGlobalVariableExternal(
          request,
          runtimeTemplate.outputOptions.globalObject
        );
      case "commonjs":
      case "commonjs2":
        return getSourceForCommonJsExternal(request);
      case "amd":
      case "amd-require":
      case "umd":
      case "umd2":
      case "system":
        return getSourceForAmdOrUmdExternal(
          chunkGraph.getModuleId(this),
          this.isOptional(moduleGraph),
          request,
          runtimeTemplate
        );
      default:
        return getSourceForDefaultCase(
          this.isOptional(moduleGraph),
          request,
          runtimeTemplate
        );
    }
  }
}
```

我们来看一下没有类型的字符串，编译后是怎么样子的。

```js
// webpack.config.js
module.exports = {
  //...
  externals: {
    lodash2: "_lodash" // 'lodash2' 表示 require 的 key，'_lodash'表示全局存在的变量
  }
  //...
};
// index.js
import Lodash from "lodash2";
Lodash.add(1, 2);
// ./dist/index.js

({
  /***/ "./index.js":
    /*!******************!*\
  !*** ./index.js ***!
  \******************/
    /*! no exports provided */
    /*! ModuleConcatenation bailout: Module is an entry point */
    /***/ function(module, __webpack_exports__, __webpack_require__) {
      "use strict";
      __webpack_require__.r(__webpack_exports__);
      /* harmony import */ var lodash2__WEBPACK_IMPORTED_MODULE_0__ = __webpack_require__(
        /*! lodash2 */ "lodash2"
      );
      /* harmony import */ var lodash2__WEBPACK_IMPORTED_MODULE_0___default = /*#__PURE__*/ __webpack_require__.n(
        lodash2__WEBPACK_IMPORTED_MODULE_0__
      );
      lodash2__WEBPACK_IMPORTED_MODULE_0___default.a.add(1, 2);
      /***/
    },

  /***/ lodash2:
    /*!**************************!*\
  !*** external "_lodash" ***!
  \**************************/
    /*! no static exports found */
    /*! ModuleConcatenation bailout: Module is not an ECMAScript module */
    /***/ function(module, exports) {
      module.exports = _lodash;

      /***/
    }

  /******/
});
```

可以看到在`require`一个 extenals 定义的 module（例子中为`lodash2`）的时候，webpack 会将`lodash2`打包为单独的一个 module，且内容为`module.exports = _lodash;`。可以看到`_lodash`其实就是一个变量，webpack 不会管，因为根据 extenals 配置，`_lodash`全局变量已经被默认是存在的。

而如果字符串中包含了特定的引入类型呢？我们继续来看另外一个例子。

```js
// webpack.config.js
module.exports = {
  //...
  externals: {
    lodash2: "root _lodash" // 'lodash2' 表示 require 的 key，'_lodash'表示全局存在的变量
  }
  //...
};
// index.js
import Lodash from "lodash2";
Lodash.add(1, 2);
// ./dist/index.js
```

#### 数组形式

数组表示从 commonjs module 中选择其导出部分的子模块。

```js
// webpack.config.js
module.exports = {
  //...
  externals: {
    substract: ["lodash2", "substract"] // require('substract') 的时候，'substract'模块的导出实际上是全局变量的 lodash 的'substract'属性
    substract2: {
      root: ["lodash2", "substract"]
    }
  }
  //...
};
// index.js
import S from "substract";
S.substract(1, 2);
// ./dist/index.js
/***/ (function(module, exports) {
  module.exports = lodash2["substract"];
  /***/
});
```

#### 对象形式

```js
module.exports = {
  externals: {
    jquery: "jQuery", //在 require('jquery') 的时候，'jquery'这个包相当于`module.exports = jQuery`，导出一个全局变量
    a: false, // 不是 external，配置错误
    b: true, // b 是 external， `module.exports = b`，适用于你所引用的库暴露出的变量和你所使用的库的名称一致的情况，等同于`b: 'b'`的情况
    "./d": "var d", // 等同于配置项为`b: 'b'`的情况
    "./f": "commonjs2 ./a/b", // "./f" 是 external `module.exports = require("./a/b")`
    "./f": "commonjs ./a/b", // ... 和 commonjs2 一样
    "./f": "this c" // module.exports = this["c"]
  }
};
```

- [webpack 中 library 和 libraryTarget 与 externals 的使用 · Issue #10 · zhengweikeng/blog](https://github.com/zhengweikeng/blog/issues/10)
- [webpack externals 深入理解 - 不长写的日志 - SegmentFault 思否](https://segmentfault.com/a/1190000012113011)
  // TODO:

更多可以查看 Webpack 官方 library[开发教程](https://webpack.js.org/guides/author-libraries/)
