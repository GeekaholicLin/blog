使用场景：

- 可以调用项目内部安装的模块，比如 `npx mocha --version` 相当于 `node-modules/.bin/mocha --version`。npx 的原理很简单，就是运行的时候，会到 `node_modules/.bin` 路径和环境变量 `$PATH` 里面，检查命令是否存在。比如 `npx ls` 相当于系统命令的 `ls`，但 `cd` 是 Bash 命令，所以无法调用
- 临时使用模块。`npx create-react-app my-react-app` 会下载到临时目录，使用以后再删除，之后再次执行命令，会重新下载 `create-react-app`。
- 执行远程 Gist 或 Github 仓库代码。远程代码必须是模块，也就是包含 `package.json` 和入口脚本。比如 Gist 代码 `npx https://gist.github.com/zkat/4bc19503fe9e9309e2bfaa2c58074d32`，比如 Github 仓库代码 `npx github:piuccio/cowsay hello`
- 可以方便地使用不同版本的 node.js 运行命令，而不需要像 `nvm` 这样的版本管理工具进行切换。比如 `npx -p node@8 node -v`

常见参数：

- `-p` 指定所安装模块，再安装多个模块时经常使用。比如 `npx -p lolcatjs -p cowsay [command]`
- `-c` 将后面字符串都用 npx 解释执行，后面的字符串就好像是调用 `npm run xxx` 一样（`npm run-scripts` 特性）。比如 `npx -p lolcatjs -p cowsay 'cowsay hello | lolcatjs'` 会报错是因为只有第一个命令 `cowsay` 会使用 npx 执行而 `lolcatjs` 会使用 Shell 解释执行。所以，为了都使用 npx，可以借助 `-c` 参数：`npx -p lolcatjs -p cowsay -c 'cowsay hello | lolcatjs'`。也可以读取 npm 的环境变量：`npx -c 'echo $npm_package_name'`
- `--no-install` 可以强制使用本地模块
- `--ignore-existing` 可以强制使用远程模块
- `--shell-auto-fallback` 会在 shell 命令找不到的时候，使用 npx 调用作为回退方案。当正确设置之后（比如 `.zshrc` 中可以加入 `npx` 插件），直接使用 `npm@4 --version` 相当于使用 `npx npm@4 --version`。常规 npx 用法和回调的不同在于，回调不安装新包，除非你使用 `pkg@version` 语法。而在本地项目中调用 `mocha --version` 相当于 `npx mocha --version`，也就是 `node-modules/.bin/mocha --version`
