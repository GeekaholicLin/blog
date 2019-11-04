理想情况下，`npm install` 应该像纯函数一样工作，对于同一个 package.json 总是生成完全相同的 node_modules 树。在某些情况下，确实如此。但在其他很多情况中，npm 无法做到这一点：

- 不同版本的 npm 的安装算法不同。
- 某些依赖项自上次安装以来，可能已发布了新版本，因此将根据 `package.json` 中的语义更新依赖。
- 某个**依赖项的依赖**可能已发布新版本，即使使用了固定依赖项说明符，它也会更新。

所以，为了在不同的环境下生成相同的 `node_modules`，npm 使用 `package-lock.json` 或 `npm-shrinkwrap.json`。

不同版本的 npm 的安装算法不同：

- npm 5.0.x 版本，不管`package.json`怎么变，`npm i` 时都会根据`package-lock.json`文件下载
- 5.1.0 版本后 `npm install` 会无视`package-lock.json`文件 去下载最新的 npm 包
- 5.4.2 版本后如果改了`package.json`，且更改版本与`package-lock.json`版本号语义不兼容，那么执行`npm i`时 npm 会根据`package.json`中的版本号以及语义去下载最新的包，并更新至`package-lock.json`。如果新修改的版本与原有的 package.json 文件中的版本兼容，那么执行`npm i `都会根据 lock 下载，不会理会`package.json`实际包的版本。

`npm install` 读取 `package.json` 创建依赖项列表，并使用 `package-lock.json` 来通知要安装这些依赖项的哪个版本

`npm ci` 是根据 `package-lock.json` 去安装确定的依赖，`package.json` 只是用来验证是不是有不匹配的版本。不匹配则报错。一般`npm ci`用于 CI 工具比如 travis 中。如果 `node_modules` 已经存在，它将在 `npm ci` 开始安装之前自动删除。`npm ci` 永远不会改变 `package.json` 和 `package-lock.json`
