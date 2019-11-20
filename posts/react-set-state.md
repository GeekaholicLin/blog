### 什么时候 `setState` 是异步的

这里所说的同步异步，并不是真正的同步异步，它还是同步执行的。这里的异步指的是多个 state 会合成到一起到最后进行**批量更新** -- 内部一般为合成事件的最后批量更新，自己手动调用`ReactDOM.unstable_batchedUpdates`则是离开此函数的时候批量更新，而导致在合成事件和钩子函数中没法立马拿到更新后的值，形式了所谓的“异步”。`setState`的参数使用函数形式可以获取最新的值。

就目前的 React v6 而言，`setState`在合成事件（以`onXxx`的方式）处理函数内部（和生命周期函数内）是异步的。

在 React 的 setState 函数实现中，会根据一个变量`isBatchingUpdates`判断是直接更新 this.state 还是放到队列中回头再说，而 `isBatchingUpdates` 默认是 `false`，也就表示 setState 会同步更新 this.state。但是，有一个函数`batchedUpdates`，这个函数会把`isBatchingUpdates`修改为 `true`，而当 React 在调用合成事件处理函数（比如`onClick`）或生命周期函数之前就会调用这个`batchedUpdates`，这就使得 setState 不会同步更新 this.state。而是批量更新。

除了合成事件处理函数内部（和生命周期函数内）是异步的，在其他函数内部也可以调用`ReactDOM.unstable_batchedUpdates`来进行强制更新，这也是`React`内部 state 批量更新的方式。

```js
promise.then(() => {
  // 强制批量更新
  ReactDOM.unstable_batchedUpdates(() => {
    this.setState({a: true}); // Doesn't re-render yet
    this.setState({b: true}); // Doesn't re-render yet
    this.props.setParentState(); // Doesn't re-render yet
  });
  // 当退出 ReactDOM.unstable_batchedUpdates 后，重新渲染一次
});
```

这确保了如果父组件和子组件在事件回调中`setState`的时候，子组件不重新渲染两次。相反地，React 会在事件结束的时候批量处理所有的 state 更新。这在大项目中是一个重要的性能提升点。在未来，React 可能会增加更多类似批量处理的场景，但目前而言只有在合成事件处理函数内部（和生命周期函数内）是异步的。

### 为什么 React 内部不同步更新 `setState`

React 内部等待所有的组件中事件处理函数的`setState`执行后，再重新渲染。这是为了避免不必要的 re-render，提升性能。

### 为什么不直接更新 `this.state`对象而是重新渲染

主要两个原因：

1. 直接更新`this.state`会破坏`props`和`state`的一致性，这会带来难以调试的问题。比如重构代码的时候，需要将状态提升，从`setState`变成`props`的回调，这就使得前后不一致。
2. 会对未来的新特性（异步渲染）实现造成很大的困难。异步更新分为「pre-commit」以及「commit」两个阶段。「pre-commit」的对比更新可能会在一次渲染周期执行多次，如果是直接更新，那会破坏组件的状态。

更详细的解释请见 [Dan 的解释](https://github.com/facebook/react/issues/11527#issuecomment-360199710)

### React 更新 state 的顺序是调用`setState`的顺序吗？

是的。

理解这个问题的关键在于，无论在多少组件的合成事件处理函数内部调用多少次`setState`，它们最终的结果只会在事件处理函数结束的时候重新渲染一次（a single re-render）。

state 的更新是按照`setState`的出现顺序进行浅合并（shallowly merged）的。比如，`{ a: 10 }`、`{ b: 20 }`以及`{ a: 30 }`，那么最终会被渲染的 state 为`{ a: 30, b: 20 }`

更多信息请查看 [相关问答](https://stackoverflow.com/questions/48563650/does-react-keep-the-order-for-state-updates)

### 参考

* [第 18 题：React 中 setState 什么时候是同步的，什么时候是异步的？ · Issue #17 · Advanced-Frontend/Daily-Interview-Question](https://github.com/Advanced-Frontend/Daily-Interview-Question/issues/17)
* [你真的理解 setState 吗？ - 掘金](https://juejin.im/post/5b45c57c51882519790c7441)
