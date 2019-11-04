---
title: 重学 React 系列 - React 16 新内容
date: 2019-10-31 18:52:00
tags: [JavaScript, React, 读书笔记]
categories: [React]
---

这里只列举零碎的知识点，关于 React Hooks 等有专门的文章进行介绍。

### 更新概览

> React v16.0：render 支持返回数组和字符串、Error Boundaries、createPortal、支持自定义 DOM 属性、减少文件体积、fiber；
> React v16.1: react-call-return；
> React v16.2：Fragment；
> React v16.3: createContext、createRef、forwardRef、生命周期函数的更新、Strict Mode；
> React v16.4: Pointer Events、update getDerivedStateFromProps；
> React v16.5: Profiler；
> React v16.6: memo、lazy、Suspense、static contextType、static getDerivedStateFromError()；
> React v16.7: React.lazy 性能问题修复
> React v16.8: React Hooks
> React v16.9: React.Profiler

更加详细的信息请见官方 [CHANGELOG](https://github.com/facebook/react/blob/master/CHANGELOG.md)

### React 16 兼容性问题

React 16 中使用了`Map`、`Set`以及`requestAnimationFrame`，如果要兼容`IE<11`的浏览器的话，需要添加相应的 polyfill

```js
import 'core-js/es/map';
import 'core-js/es/set';
import 'raf/polyfill';

import React from 'react';
import ReactDOM from 'react-dom';

ReactDOM.render(
  <h1>Hello, world!</h1>,
  document.getElementById('root')
);
```

### ReactDOM.findDOMNode 的变化

`ReactDOM.findDOMNode(componentInstance)`用于通过组件的实例得到相对应的 DOM 元素。需要注意的是，`findDOMNode`只能用于已经挂载的组件（mounted component）。

在 React 16 引入`fragment`的概念后，`findDOMNode`在遇到`fragment`的时候会返回第一个非空子元素（the first non-empty child）。

### React16 前后的生命周期对比，以及需要注意的点

![React 15 生命周期](http://image.geekaholic.cn/20191030143834.png@0.8)
![React 16 生命周期](http://image.geekaholic.cn/20191030144005.png@0.8)

React 16 的生命周期有两个版本，React 16.3 版本以及之后的版本。React 16.3 版本中，`setState`以及`forceUpdate`并不会触发`getDerivedStateFromProps`，而是直接达到下一个的`shouldComponentUpdate`（对于`setState`而言）或`render`（对于`forceUpdate`而言），在图中的体现就是，`getDerivedStateFromProps`的长度要短。可以查看 [React 生命周期图](http://projects.wojtekmaj.pl/react-lifecycle-methods-diagram/) 进行对比。

即将被废弃的生命周期有`componentWillMount()`（基本没什么用，初始化在`constructor`，挂载后的请求、DOM 操作逻辑在`componentDidMount`）、`componentWillReceiveProps(nextProps)`（坑太多，特别是重新渲染的时候数据更新问题，详情请见 [blog](https://reactjs.org/blog/2018/06/07/you-probably-dont-need-derived-state.html)）、`componentWillUpdate`（基本没什么应用场景）

新增的生命周期有平常渲染相关的`static getDerivedStateFromProps()`与`getSnapshotBeforeUpdate`, 和错误处理相关的`static getDerivedStateFromError()`与`componentDidCatch`。

#### shouldComponentUpdate(nextProps, nextState)

之前一直对此方法有认知错误。在父级组件`shouldComponentUpdate`的生命周期函数返回`false`，其下边的子组件依然**有可能**更新。一个组件是否 re-render 是看它自身的 state 或 props 是否改变（或者说是否强制调用 this.forceUpdate），与其他组件没有关系。要想子组件不更新，应该也要在`shouldComponentUpdate`中进行逻辑处理并返回`false`

#### static getDerivedStateFromProps(props, state)

目的在于当 props 改变的时候，返回的 state 对象用于更新其内部 state（返回 null 的时候不更新）。从 React 16 生命周期图中可以看出，在每次 render 前都会触发该方法，也就是包括 初始化、`state`改变、`props`改变以及`forceUpdate`。但也会有很多坑，原因与`componentWillReceiveProps(nextProps)`一样。

一般来说，此方法的运用场景有其他的替代方式，详情请见 [blog](https://reactjs.org/blog/2018/06/07/you-probably-dont-need-derived-state.html)：

- 如果是依据 props 改变的副作用操作，比如数据获取或动画效果，可以将逻辑放入`componentDidUpdate`
- 如果是只有 props 改变才重新计算或重新渲染，可以使用记忆化 (`memorize`) 技术
- 如果是依据 props 的改变更改内部的 state，可以考虑转换成控制组件（ fully controlled）或使用 key 强制刷新的非控制组件（fully uncontrolled with a key）

与`componentWillReceiveProps`的不同在于，`getDerivedStateFromProps`内部无法获取当前组件的实例，因为是静态方法。而且每次 render 前都会执行`getDerivedStateFromProps`，而`componentWillReceiveProps`只会在 props 改变的时候

#### getSnapshotBeforeUpdate(prevProps, prevState)

在预提交阶段（`pre-commit phase`）执行，可以获取 DOM 元素。此生命周期函数的返回值会作为`componentDidUpdate(prevProps, prevState, snapshot)`的第三个参数。较为常见的场景是用于记录**当前组件**的滚动位置。

#### static getDerivedStateFromError(error)

在**子组件**抛出异常的时候执行，其中捕获的 error 作为参数，且应该返回一个新的 state 用于指示错误已经发生，用于渲染新的 UI。

```js
class ErrorBoundary extends React.Component {
  state = { hasError: false }
  static getDerivedStateFromError(error) {
    // Update state so the next render will show the fallback UI.
    return { hasError: true };
  }
  render() {
    if (this.state.hasError) {
      // You can render any custom fallback UI
      return <h1>Something went wrong.</h1>;
    }
    return this.props.children;
  }
}
```

需要注意的是，`getDerivedStateFromError`是在「render 阶段」执行的，所以最好不要用于执行副作用。可以把它当成「接收错误-返回错误状态的 state」纯函数，根据不同的错误或状态码派生出不同的 state 数据，从而渲染出不同的 UI。至于副作用操作，可以移到`componentDidCatch`中进行处理。

#### componentDidCatch(error, info)

在**子组件**抛出异常的时候执行，执行阶段是在「commit 阶段」执行。其中参数 error 为捕获的错误对象，info 是一个只有`componentStack`属性，记录的是报错组件堆栈信息。在抛出异常所需的副作用，比如日志打印等，可以在此生命周期函数中执行。

在`componentDidCatch`中可以执行副作用，当然也可以执行`setState`对 UI 进行更新。但是官方并不推荐在`componentDidCatch`中执行`setState`，因为未来可能会不支持这种做法，且已经有相关的生命周期函数`static getDerivedStateFromError()`帮忙处理。

### Fragments

用于解决 render 函数中返回数组时，一定要有外层包装的问题，同时不引入额外的标签。目前只有支持`key`属性。

```js
class Table extends React.Component {
  render() {
    return (
      <table>
        <tr>
          <Columns />
        </tr>
      </table>
    );
  }
}
class Columns extends React.Component {
  render() {
    return (
      <>
        <td>Hello</td>
        <td>World</td>
      </>
    );
  }
}
```

### Protals

```js
// 语法，表示在 container 内部渲染 child
ReactDOM.createPortal(child, container)

// 使用
render() {
  // React does *not* create a new div. It renders the children into `domNode`.
  // `domNode` is any valid DOM node, regardless of its location in the DOM.
  return ReactDOM.createPortal(
    this.props.children,
    domNode
  )
}
```
React Protals 是为了在渲染子组件时突破 Component 树结构的 DOM 层级限制，但又保留了事件冒泡、Context 等 React 逻辑层级上的相关处理

### Error boundaries

局部 UI 的 JavaScript 错误不应该导致整个应用的崩溃，为了解决这个问题，React 引入了「错误边界」的概念。错误边界是一个组件，负责捕获子组件的异常、处理异常并显示回退方案的 UI。错误边界捕获的错误是子组件在 render 阶段、生命周期函数以及构造函数的错误，这就意味着错误边界**不能捕获**事件处理中的错误、异步代码中的错误、服务端渲染中的错误以及自身的错误。

```jsx
// 定义
class ErrorBoundary extends React.Component {
  constructor(props) {
    super(props);
    this.state = { hasError: false };
  }

  static getDerivedStateFromError(error) {
    // Update state so the next render will show the fallback UI.
    return { hasError: true };
  }

  componentDidCatch(error, errorInfo) {
    // You can also log the error to an error reporting service
    logErrorToMyService(error, errorInfo);
  }

  render() {
    if (this.state.hasError) {
      // You can render any custom fallback UI
      return <h1>Something went wrong.</h1>;
    }

    return this.props.children;
  }
}
// 使用
<ErrorBoundary>
  <MyWidget />
</ErrorBoundary>
```

### Strict Mode

用于检测其子组件中不规范的使用并给出警告，`React.StrictMode`本身不会进行渲染 DOM 且只在 dev 环境下有效。

不规范的使用包括：

- 使用 React 16 之后不安全的生命周期函数，比如 React 16 之后异步渲染可能导致多次调用的`componentWillMount`等
- 使用 string 类型的 ref
- findDOMNode 的使用
- 意料之外的副作用：在即将到来的异步渲染中，render 阶段的生命周期函数可能会被执行多次，所以不应该在 render 阶段的生命周期中执行副作用。但是这些生命周期中的副作用很难检测。而 String Mode 可以帮助检测得稍微容易一点，具体做法就是对`constructor`、`render`、`setState`的 updater functions 以及`static getDerivedStateFromProps`执行两次。
- 使用旧的 context API

### React.lazy 与 React.Suspense

`React.lazy`用来接收一个返回 Promise 的函数，用于动态加载组件。这个 Promise 会在网络请求成功后，resolve 加载的模块对象，其中`default`属性指向这个加载的模块。

而`React.Suspense`其`fallback`属性接收 React element，用于作为在加载过程中的加载渲染组件。`React.Suspense`组件必须在动态加载组件之上，且可以包裹多个。一般`React.Suspense`用于需要显示 loading 加载指示 UI 的地方。

目前`React.lazy`不支持服务端渲染的代码分割且不支持`import(`./${value}`)`的。官方推荐使用`loadable-component`库。

`React.lazy`也可以和 React 中的`Error boundaries`进行配合使用，一般来说用于包裹动态加载组件。

```js
import MyErrorBoundary from './MyErrorBoundary';
const OtherComponent = React.lazy(() => import('./OtherComponent'));
const AnotherComponent = React.lazy(() => import('./AnotherComponent'));

const MyComponent = () => (
  <div>
    <MyErrorBoundary>
      <Suspense fallback={<div>Loading...</div>}>
        <section>
          <OtherComponent />
          <AnotherComponent />
        </section>
      </Suspense>
    </MyErrorBoundary>
  </div>
);

```

### React.memo

`React.memo`的功能与`React.PureComponent`一样，都是用于浅比较决定是否更新。不同的是前者用于函数组件，后者用于类组件。

```js
// https://github.com/facebook/react/blob/769b1f270e1251d9dbdce0fcbd9e92e502d059b8/packages/react/src/memo.js
// 简化后的代码
import {REACT_MEMO_TYPE} from 'shared/ReactSymbols';
export default function memo<Props>(
  type: React$ElementType,
  compare?: (oldProps: Props, newProps: Props) => boolean,
) {
  return {
    $$typeof: REACT_MEMO_TYPE,
    type,
    compare: compare === undefined ? null : compare,
  };
}
```

从上边简化后的代码而言，`React.memo`接收一个 component（只能是函数组件）以及一个比较函数（默认是浅比较），当比较函数返回 true 的时候表示前后一样，不进行 UI 的更新。注意这里与`shouldComponentUpdate`刚好相反，比较函数结果为 true 表示不需要更新，而`shouldComponentUpdate`返回 true 表示应该更新。

### Context

React 中的数据传递是从父组件**逐级**传递到下边的子组件，当**层级较深且许多组件需要**的时候（比如国际化、主题等），props 的传递会显得繁琐。所以引入了 Context，方便在组件之间共享而不需要显式地逐级传递。

Context 的适用场景是**层级较深且许多组件需要**，如果是层级较深但只有较深层级需要对应的 props，可以优先考虑「控制反转」 -- 也就是将深层级的组件（假设为`ChildComponent`）作为父级的 props（假设这里为`sidebar`）进行下传，这样可以减少传递的 props 层级（有点像 Vue 中的 slot）。除此之外，如果深层级的组件需要父级组件的一些数据作为该组件的 props，则可以使用 render props 技术，将`sidebar`改造成返回`ChildComponent`的函数。

```jsx
// 以 'light' 为默认值创建 context 对象
const ThemeContext = React.createContext('light');
// 在 DevTools 中显示`ThemeContext.Provider`或`ThemeContext.Consumer`
ThemeContext.displayName = 'ThemeContext'

class App extends React.Component {
  render() {
    // context 对象提供的值为 value 属性指定的值，也就是 'dark'
    return (
      <ThemeContext.Provider value="dark">
        <Toolbar />
      </ThemeContext.Provider>
    );
  }
}

// 不需要显式传递
function Toolbar(props) {
  return (
    <div>
      <ThemedButton />
      <ThemeContext.Consumer>
        {value => <Button theme={value} />}
      </ThemeContext.Consumer>
      <ThemeContext.Consumer>
        {value => <div>{value}</div>}
      </ThemeContext.Consumer>
    </div>
  );
}

class ThemedButton extends React.Component {
  // 用 static contextType 指定对应的 context 对象，这里为 ThemeContext
  // React 会往上找到最近一个对应的 context 对象，然后使用它的 value 值
  // 这里 this.context 为 'dark'
  static contextType = ThemeContext;
  render() {
    return <Button theme={this.context} />;
  }
}
```

以 `light` 为默认值创建 Context 对象，默认值的作用在于当子组件中没有匹配到 Provider 才会生效。这有助于在不实用 Provider 包装子组件的时候对组件进行测试。

Provider 接收一个 value 属性，传递给消费组件（`ThemeContext.Consumer`组件或`static contextType`）。一个 Provider 可以和多个消费组件有对应关系。多个 Provider 也可以嵌套使用，里层的会覆盖外层的数据。

当 Provider 的 value 值发生变化时（使用与`Object.is`相同的算法进行比较），它内部的所有消费组件都会重新渲染（所以特别注意，不要给 value 赋值为对象字面量或者匿名箭头函数，因为这样 value 总为新值）。Provider 及其内部 consumer 组件都**不受制于** shouldComponentUpdate 函数。

使用`static contextType`可以指定当前组件中`this.context`对应的 context 对象。`this.context`可以在任意生命周期函数中访问，包括 render 以及 constructor（但是需要在`constructor(props, context){}`中使用第二个参数，也就是`super(props, context)`）。

除了`static contextType`外，还可以使用`ThemeContext.Consumer`进行订阅 context 对象的值。两者的区别在于`ThemeContext.Consumer`借助了 render props 的特性，更加地灵活。比如在订阅多个 context 对象的时候：

```jsx
<ThemeContext.Consumer>
  {theme => (
    <UserContext.Consumer>
      {user => <ProfilePage user={user} theme={theme} />}
    </UserContext.Consumer>
  )}
</ThemeContext.Consumer>
```

---

参考：
* [React16 新特性 - 掘金](https://juejin.im/post/5c0e1583e51d45780317b32a)