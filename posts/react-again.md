---
title: 重学 React 系列 - React 15 的相关回顾
date: 2019-10-31 18:51:00
tags: [JavaScript, React, 读书笔记]
categories: [React]
---

### reconciliation 的概念

> When a component’s props or state change, React decides whether an actual DOM update is necessary by comparing the newly returned element with the previously rendered one. When they are not equal, React will update the DOM. This process is called “reconciliation”.

当组件的 props 或 state 发生变化，React 将前后元素进行对比决定是否更新。如果不等则进行 DOM 的更新。这个处理过程称为`reconciliation`。

协调处理过程中包括了 Virtual DOM 的 diff 算法。在 React 16 之前，协调过程或协调算法称为`stack reconciliation`。从 React 16 开始，采用了新的增量式协调引擎`Fiber`，相应的协调过程被称为`fiber reconciliation`。

### Shadow DOM 与 Virtual DOM

Virtual DOM 是内存中对 UI 的对象表示。在 React 中，Virtual DOM 与 React elements 相关联，因为后者在 React 中也是使用对象对 UI 进行表示。但是 React 中还使用了`fibers`的内部对象给组件树赋予了更多的属性。

Shadow DOM 与 Virtual DOM 不是相同的东西。Shadow DOM 是浏览器的标准，用于在 Web Component 中对 HTML 与 CSS 的封装，解决 CSS 等污染问题。而 Virtual DOM 是在浏览器之上，对 UI 的一个虚拟概念的表示。

### 应该在 React 组件的哪个生命周期发起 Ajax 请求

`componentDidMount`。相对于`componentWillMount`而言，一是因为这样可以确保 Ajax 请求之后，组件已经挂载。二是因为这只会执行一次，如果在`componentWillMount`中可能执行多次，比如 SSR（同构） 的场景，一次在服务器端执行一次在客户端执行。在 React 16 之后引入 Fiber 的概念，`componentWillMount`、`componentWillReceiveProps`、`shouldComponentUpdate`以及`componentWillUpdate`可能被调用不止一次，且后续这些生命周期将被废弃。

### forceUpdate 的作用是什么

在 React 中，默认情况下如果 state 或者 props 发生变化，组件会重新渲染。但是如果依赖了其他数据（非 state 以及 props），当数据变化或需要手动强制更新的时候，需要用`React.forceUpdate()`告诉 React 强制重新渲染。这会跳过`shouldComponentUpdate()`生命周期的判断，直接调用 render 以及 render 之后的生命周期，可以查看 React 的生命周期图。另外，`React.forceUpdate()`只会作用当前组件，至于子组件会不会重新渲染（re-render），则看看子组件的`shouldComponentUpdate()`的判断，相当于子组件重新走一遍更新的生命周期流程。

### PureComponent 与 Component 的区别

PureComponent 从效果上看是类似实现了浅比较的`shouldComponentUpdate`的 Component。但实际上 PureComponent 继承自 Component，中间借助了`ComponentDummy`来进行实现

```js
function ComponentDummy() {}
ComponentDummy.prototype = Component.prototype;
const pureComponentPrototype = (PureComponent.prototype = new ComponentDummy());
pureComponentPrototype.constructor = PureComponent;
// Avoid an extra prototype jump for these methods.
Object.assign(pureComponentPrototype, Component.prototype); // 将方法下移，减少寻找层级
pureComponentPrototype.isPureReactComponent = true;
```

React 中用不同的变量来标示 Component 与 PureComponent。如果是 Component 则`React.isReactComponent`为`{}`，而 PureComponent 则使用`isPureReactComponent`为`true`且`React.isReactComponent`为`{}`（因为是继承关系）

### 什么是受控组件

> An input form element whose value is controlled by React is called a controlled component.
> An uncontrolled component works like form elements do outside of React.
> In most cases you should use controlled components.

在 React 中，所谓受控组件和非受控组件，是针对表单而言的。

受控组件是指，让 React 通过`onChange`等事件触发，更新`state`的值继而控制那些会维持自身的状态的表单组件，比如`input`、`textarea`等。如果只使用`value={this.state.value}`而没有相关的 handler，React 将会抛出警告。

而非受控组件指的是不受状态的控制，从 DOM 中直接获取数据。

在 React 中大多数情况下应该使用受控组件。

### React 中 key 的作用是什么

当创建数组类型的元素时，key 属性能帮助 React 识别哪些元素被新增、删除还是被移动，对元素尽可能地复用。所以 key 属性应该是**唯一且稳定**的标示。不唯一会使得除了第一个 key 外的其他相同 key 的元素不渲染，不稳定（每次 re-render 都不一样）会使得 React 不断地「销毁-重建」，导致性能问题。

key 不需要当下全局地问题，只需要在当前数组唯一即可。

### 在构造函数中调用 super(props) 的目的是什么？

- 在继承中，如果写`constructor`构造函数，必须写`super()`，进行父类构造函数的调用。
- 在`super()`之前，无法使用`this`关键字，这是 JS 的规范
- 为什么要传`props`？是因为要让`React.Component`进行初始化`this.props`（在`Component`内部，`this.props = props;`），能在**构造函数**中使用`this.props`。不过即使不传`props`，也可以在**构造函数**使用`props`（这是不推荐的），在**其他生命周期函数**使用`this.props`。
- 为什么直接使用`super()`在其他生命周期也能调用`this.props`？这是因为 React 在**调用构造函数之后**，马上又帮忙设置了一次`props`

```js
// React 内部帮忙的处理
const instance = new YourComponent(props);
instance.props = props; // 感觉作用更像是「复制」一份到子类的实例属性上
```

- 所以，即使`super(props)`的`props`非必要（因为构造函数可以通过`props`访问，其他生命周期函数有 React 内部帮忙处理），但是官方依然推荐**总是**使用`super(props)`，考虑以下情况，可能很难去定位问题：

```js
class Button extends React.Component {
  constructor(props) {
    super(); // 😬 我们忘了传入 props
    console.log(props);      // ✅ {}
    console.log(this.props); // 😬 undefined
  }
}
```

### Element 以及 Component 的区别

`Element`用于描述在屏幕中显示的 UI，比如`<h1 className="greeting">Hello, world</h1>`。但在 React 中，React element 与浏览器原生的 DOM element 不同，React element 是原生对象。JSX 语法描述的 UI 会被 Babel 编译成调用`React.createElement('h1', {className: 'greeting'}, 'Hello, world')`，而此方法会返回一个对象用于描述 UI：

```js
{
  type: 'h1',
  props: {
    className: 'greeting',
    children: 'Hello, world!'
  },
  $$typeof: Symbol.for('react.element')
  //...
}
```

> Conceptually, components are like JavaScript functions. They accept arbitrary inputs (called “props”) and return React elements describing what should appear on the screen.

而`component`则是在概念上相当于一个 JS 函数，接收任意多个参数（也就是`props`），并返回 React element。

### React 中 props 是如何禁止修改的

利用了`Object.freeze()`，不过只是浅层的冻结。

```js
// https://github.com/facebook/react/blob/d3622d0f977def825123f1d5f4cef19888b1eaf1/packages/react/src/ReactElement.js#L158-L161
if (Object.freeze) {
  Object.freeze(element.props);
  Object.freeze(element);
}
```

### React 中合成事件与 DOM 原生事件的区别

- React 中驼峰命名而原生全小写，且值在 React 中为函数而在原生中是字符串

```js
// 原生
<button onclick="activateLasers()">Activate Lasers</button>
// react
<button onClick={activateLasers}>Activate Lasers</button>
```

- 阻止默认行为的时候，原生可以返回`false`，而 React 只能使用`e.preventDefault()`

```js
// 原生
<a href="#" onclick="console.log('The link was clicked.'); return false">
  Click me
</a>
// react
function ActionLink() {
  function handleClick(e) {
    e.preventDefault();
    console.log('The link was clicked.');
  }
  return (
    <a href="#" onClick={handleClick}>
      Click me
    </a>
  );
}
```

### createElement 与 createFactory

`React.createElement`是 JSX 语法糖背后调用的函数，用于生成元素，语法为：

```js
React.createElement(
  type,
  [props],
  [...children]
)

// <h1 className='greeting'>Hello, world</h1>
// React.createElement('h1', {className: 'greeting'}, 'Hello, world')
```

而`React.createFactory`其实是`React.createElement`的包装，通过工厂方法创建 React 组件的实例。

```js
ReactElement.createFactory = function(type) {
  var factory = ReactElement.createElement.bind(null, type);
  return factory;
};
```

### element 操作相关

#### React.cloneElement

```js
// 语法
React.cloneElement(
  element,
  [props],
  [...children]
)
// <element.type {...element.props} {...props}>{children}</element.type>
```

可以看到语法和`React.createElement`如出一辙，但是有几点不同。一是 props 不是替代模式而是合并模式，会将`React.cloneElement`的 props 与原来元素的 props 进行合并。二是从 props 中或 children 的 props 中拿到相应的 ref 属性。

#### React.isValidElement(obj)

判断传入的对象是否是 React element

#### React.Children 相关

如果子元素`children`是`Fragment`类型元素，则`children`被当成单个元素。

- `React.Children.map`：遍历`children`，返回新的。如果子元素为`null`或者`undefined`则返回`null`或`undefined`，不会返回空数组。
- `React.Children.forEach`：遍历`children`，但不返回结果
- `React.Children.count`：返回`children`的组件个数
- `React.Children.only`：判断`children`是否单个元素，不是的话抛出异常
- `React.Children.toArray`：将`children`转换成数组，并且添加`key`属性。如果`key`属性已经存在会在前面自动添加 prefix
