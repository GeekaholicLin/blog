# React 合成事件

## 入门

### 简介

> 简介部分来自官方文档

React 并没有使用原生的浏览器事件，而是在基于 Virtual DOM 的基础上实现了合成事件 (`SyntheticEvent`)，事件处理程序接收到的是 `SyntheticEvent` 的实例。

`SyntheticEvent` 完全符合 W3C 的标准，因此在事件层次上具有浏览器兼容性，与原生的浏览器事件一样拥有同样的接口，可以通过 `stopPropagation()` 和 `preventDefault()` 相应的中断。如果需要访问当原生的事件对象，可以通过引用 `nativeEvent` 属性获得。
`SyntheticEvent`对象拥有以下属性：

```plain
boolean bubbles
boolean cancelable
DOMEventTarget currentTarget
boolean defaultPrevented
number eventPhase
boolean isTrusted
DOMEvent nativeEvent
void preventDefault()
boolean isDefaultPrevented()
void stopPropagation()
boolean isPropagationStopped()
DOMEventTarget target
number timeStamp
string type
```

为了性能，React 的合成事件引入了事件池「Event Pooling」的概念，这意味着复用`SyntheticEvent`且在事件回调执行完成后置为空。所以，这里容易踩坑，就是会在异步中无法获取`SyntheticEvent`实例，比如`this.setState({clickEvent: event});`会得到`{clickEvent: null}`。如果一定要在异步中获取合成事件`event`对象，可以使用`event.persist()`方法，可以将合成事件从事件池中移除而使得相关的值得以保留。

```js
hanleClick = event => {
  event.persist(); // 显式调用
  setTimeout(() => {
    this.setState({
      search: event.target.value // 正常获取
    });
  });
};
```

React 实现了很多的合成事件，默认情况下合成事件的触发是「冒泡阶段」。如果想在「捕获阶段」执行相关回调逻辑，可以在合成事件后面添加`Capture`，比如在冒泡阶段触发的`onClick`后添加变成在捕获阶段触发的`onClickCapture`。

### 意义

为什么需要自定义一套事件系统？

- 抹平浏览器之间的兼容性差异
- 统一事件：比如`onChange`事件统一了表单元素的值变动事件
- 抽象跨平台事件机制：与 VirtualDOM 的意义类似，目的在于提供抽象的跨平台事件机制
- 优化：利用事件委托机制以及对象池管理合成事件对象的创建和销毁，减少内存开销
- 优先级与可中断：为了 React 未来的「异步渲染」新特性，让不同类型的事件有不同的优先级，且可断。

### 机制

React 事件机制分为「事件注册」和「事件分发」。其中「事件注册」将所有的事件都委托代理到`document`（的冒泡阶段），且每一种类型的事件都拥有统一的`dispatchEvent`回调函数，不用来处理具体事件，仅仅是用来执行事件分发；而「事件分发」**自身实现了一套冒泡机制**，会在相应类型事件的`dispatchEvent`回调中先生成合成事件（通过对象池管理合成事件的创建及销毁），根据触发的`target`将事件分发到具体的组件实例，再不断地从触发组件向父组件**回溯**，依次调用相应事件类型的 callback，实现类似冒泡的效果。

### 混用使用

> event.stopPropagation 阻止捕获和冒泡阶段中**当前类型事件**的进一步传播。

> 如果有**多个相同类型事件**的事件监听函数绑定到同一个元素，当该类型的事件触发时，它们会按照被添加的顺序执行。如果其中某个监听函数执行 `event.stopImmediatePropagation` 方法，则当前元素剩下的**相同类型事件**监听函数将不会被执行，且该类型事件不会冒泡。

为了深入理解原生事件回调和合成事件回调在 React 中的执行顺序以及`stopPropagation()`与`stopImmediatePropagation()`分别在合成事件与原生事件的作用，以下面代码为例子进行说明：

```js
class App extends React.Component {
  componentDidMount() {
    document
      .getElementById("child")
      .addEventListener("click", function clickNativeChildOne(e) {
        // e.stopPropagation() // case 2
        // e.stopImmediatePropagation() // case 4
        console.log("child native click one");
      });
    document
      .getElementById("child")
      .addEventListener("click", function clickNativeChildTwo(e) {
        console.log("child native click two");
      });
    document
      .getElementById("parent")
      .addEventListener("click", function clickNativeParent() {
        console.log("parent native click");
      });
    document.addEventListener("click", function clickNativeDocument(e) {
      console.log("document native click");
    });
  }
  handleChildClick = e => {
    // e.stopPropagation() // case 1
    // e.nativeEvent.stopImmediatePropagation() // case 3
    console.log("child synthetic click");
  };

  handleParentClick = e => {
    console.log("parent synthetic click");
  };

  render() {
    return (
      <div id="parent" onClick={this.handleParentClick}>
        <div id="child" onClick={this.handleChildClick} />
      </div>
    );
  }
}
// 直接点击 child 元素
// "child native click one"
// "child native click two"
// "parent native click"
// "child synthetic click"
// "parent synthetic click"
// "document native click"

// 1. handleChildClick 中使用 e.stopPropagation()
// "child native click one"
// "child native click two"
// "parent native click"
// "child synthetic click"
// "document native click"

// 2. clickNativeChildOne 中使用 e.stopPropagation()
// "child native click one"
// "child native click two"

// 3. handleChildClick 中使用 e.nativeEvent.stopImmediatePropagation()
// "child native click one"
// "child native click two"
// "parent native click"
// "child synthetic click"
// "parent synthetic click"

// 4. clickNativeChildOne 中使用 e.stopImmediatePropagation()
// "child native click one"
```

直接点击 child 元素的时候，会先执行原生事件，原生事件一直冒泡并执行相应回调，一直到 document。而原生 document 的 click 事件后于合成事件的`dispatchEvent`回调，这是因为 document 对合成的 click 事件监听初始化就添加了（相当于`document.addEventListener('click'， dispatchEvent)`），相对于 componentDidMount 再添加的原生 click 事件监听更早。

情况 1，在`handleChildClick`中使用`e.stopPropagation()`，此时的`e`为合成事件对象，不会阻止原生事件，只会阻止 React 的合成事件，所以`"parent native click"`没有打印。

情况 2，在`clickNativeChildOne`中使用`e.stopPropagation()`，此时的`e`为原生事件对象，会阻止原生事件冒泡，进而影响了`document`上`dispatchEvent`的执行。

情况 3，在`handleChildClick`中使用`e.nativeEvent.stopImmediatePropagation()`，此时的`e.nativeEvent`为原生事件对象，因为`handleChildClick`执行的时候已经是在`document`节点的`dispatchEvent`回调，`stopImmediatePropagation`阻止的是`document`节点后续注册的`click`监听，所以`"document native click"`没有打印。

情况 4，在`clickNativeChildOne`中使用`e.stopImmediatePropagation()`，此时的`e`为原生事件对象，会阻止 click 事件的冒泡，以及后续相同 click 事件回调的执行，所以只会打印`"child native click one"`

## 源码深入（TODO:）

- [动画浅析 React 事件系统和源码](https://www.lzane.com/tech/react-event-system-and-source-code/)
- [React 合成事件源码解读，还原背后辛酸史](https://juejin.im/post/5d43d7016fb9a06aff5e5301)
- [Caylor](https://www.caylor.cc/post/2018-06/react%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB-4-%E5%90%88%E6%88%90%E4%BA%8B%E4%BB%B6/)
- [谈谈 React 事件机制和未来 (react-events)](https://juejin.im/post/5d44e3745188255d5861d654)

## 参考

- [React 事件代理与 stopImmediatePropagation](https://github.com/youngwind/blog/issues/107)
