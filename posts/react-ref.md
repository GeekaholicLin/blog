---
title: 重学 React 系列 - React Ref
date: 2019-10-29 20:48:30
tags: [JavaScript, React]
categories: [React]
---

### react-ref 的创建方式

你不能在函数**组件上**使用 ref 属性，因为他们没有实例。

- `string ref`（不推荐），原因见下方相关章节。函数组件**内部**不支持使用`string ref`
- `callback ref`
- `React.createRef()` - React v16.3
- Hooks `React.useRef()` - React v16.8

### React.createRef 与 React.forwardRef

```js
// React.createRef 源代码
import type {RefObject} from 'shared/ReactTypes';

// an immutable object with a single mutable value
export function createRef(): RefObject {
  const refObject = {
    current: null,
  };
  if (__DEV__) {
    Object.seal(refObject);
  }
  return refObject;
}
// React.forwardRef 源代码简化版
import {REACT_FORWARD_REF_TYPE} from 'shared/ReactSymbols';

import warningWithoutStack from 'shared/warningWithoutStack';

export default function forwardRef<Props, ElementType: React$ElementType>(
  render: (props: Props, ref: React$Ref<ElementType>) => React$Node,
) {
  return {
    $$typeof: REACT_FORWARD_REF_TYPE,
    render,
  };
}
```

`React.createRef()`的作用是创建并返回具有`current`属性的对象，在`ComponentDidMount`或`ComponentDidUpdate`的时候对应 DOM 节点或 Component 实例挂载在`current`属性上，而`React.forwardRef(render)`接收一个函数`render`，`render`函数接收`props`以及`ref`，并返回一个`Forward`类型的 React 元素的描述对象。

```jsx
// 单独使用 React.createRef
// 例子来自官网
class AutoFocusTextInput extends React.Component {
  constructor(props) {
    super(props);
    this.textInput = React.createRef();
  }

  componentDidMount() {
    this.textInput.current.focusTextInput();
  }

  render() {
    return (
      <CustomTextInput ref={this.textInput} />
    );
  }
}
```

要从父组件获取子组件或后代的`ref`（Forwarding Refs），除了常见的使用`callback ref`以及`props callback`结合的机制，在 React v16.3 之后还可以使用`React.createRef()`以及`React.forwordRef()`结合的方法。

```jsx
// 来自官网的简单例子
const FancyButton = React.forwardRef((props, ref) => (
  <button ref={ref} className="FancyButton">
    {props.children}
  </button>
));

// You can now get a ref directly to the DOM button:
const ref = React.createRef();
<FancyButton ref={ref}>Click me!</FancyButton>;

```

需要注意的是，只有通过`React.forwordRef()`创建包裹的组件才会接收`ref`属性作为第二个参数。所以`React.forwordRef()`的用途只是提供接收`ref`参数的功能，与`React.createRef()`并不是强绑定，但是两者一般一起使用。`React.forwordRef()`接收`ref`作为第二参数的好处在于，可以方便在 HOC 中传递 ref。在没有`React.forwordRef()`之前，想获取 HOC 被包装组件内部某个组件实例或元素，只能通过`getInnerRef={(childInput) => this.childInputRef = childInput}`这种重新命名的形式。

```js
// 例子来自官方文档
function logProps(Component) {
  class LogProps extends React.Component {
    render() {
      const {forwardedRef, ...rest} = this.props;
      // return <Component {...this.props} />
      return <Component ref={forwardedRef} {...rest} />;
    }
  }
  // return LogProps
  const forwardRefRender = (props, ref) => {
    return <LogProps {...props} forwardedRef={ref} />;
  }
  const name = Component.displayName || Component.name;
  forwardRefRender.displayName = `logProps(${name})`; // 添加名字，便于调试
  return React.forwardRef(forwardRefRender);
}
class OriginalButton extends React.Component {
  focus() {
    // ...
  }
}
const ref = React.createRef();
const FancyButton = logProps(OriginalButton)
<FancyButton
  label="Click Me"
  handleClick={handleClick}
  ref={ref}
/>
ref.current.focus() // 调用 OriginalButton 实例的方法
```

### React.useRef

```js
// 官方例子
function TextInputWithFocusButton() {
  // const inputEl = useRef(initialValue);  // { current: initialValue }
  const inputEl = useRef(null);
  const onButtonClick = () => {
    // `current` points to the mounted text input element
    inputEl.current.focus();
  };
  return (
    <>
      <input ref={inputEl} type="text" />
      <button onClick={onButtonClick}>Focus the input</button>
    </>
  );
}
```

`React.useRef`的本质作用更像是一个实例的挂载点。它返回的对象就像是与实例相挂钩，返回的`{ current: initialValue }`除了被密封（seal）导致无法拓展（non-extensible）以及无法配置
（non-configurable）外，与`const inputEl = { current: null }`的区别在于，这个对象在重新渲染后依然是相同的引用。

利用它的本质作用，其实不仅仅可以在 DOM 元素 mounted 的时候存储 DOM 元素的引用，还可以当成实例的相关变量（instance variables in a class），就好像在使用`this.inputEl`一样。

```js
// 官方例子
function Timer() {
  const intervalRef = useRef();

  useEffect(() => {
    const id = setInterval(() => {
      // ...
    });
    intervalRef.current = id; // 存储，可以方便在外部使用
    return () => {
      clearInterval(intervalRef.current);
    };
  });

  // ...
}

```

### 为什么 ref 不推荐使用 string ref

- 一定的性能问题。当为 string ref 的时候，React 需要追踪**正在渲染**的组件，将 ref 包装成闭包在 commit 阶段进行执行，对性能有一定的影响

```typescript
// React.Component uses a shared frozen object by default.
// We'll use it to determine whether we need to initialize legacy refs.
export const emptyRefsObject = new React.Component().refs;
// https://github.com/facebook/react/blob/e0a521b02ad54b840ee66637f956b65db4dbe51c/packages/react-reconciler/src/ReactChildFiber.js#L180-L193
// 转换 string ref 为 function 形式
function coerceRef(
  returnFiber: Fiber,
  current: Fiber | null,
  element: ReactElement
) {
  let mixedRef = element.ref;
  const owner: ?Fiber = (element._owner: any);
  let inst;
  if (owner) {
    const ownerFiber = ((owner: any): Fiber);
    inst = ownerFiber.stateNode;
  }
  const stringRef = "" + mixedRef;
  // Check if previous string ref matches new string ref
  const ref = function(value) {
    let refs = inst.refs;
    if (refs === emptyRefsObject) {
      // This is a lazy pooled frozen object, so we need to initialize.
      refs = inst.refs = {};
    }
    if (value === null) {
      delete refs[stringRef];
    } else {
      refs[stringRef] = value;
    }
  };
  ref._stringRef = stringRef;
  return ref;
}
```

- 当使用 render callback 模式的时候，使用 string ref 会造成 ref 挂载位置产生歧义。

```js

class MyComponent extends Component {
  renderRow = (index) => {
    // string ref 会挂载在 DataTable this 上
    // 因为返回的闭包，this 指向实际上是 renderRow 函数调用所在上下文
    return <input ref={'input-' + index} />;

    // callback ref 会挂载在 MyComponent this 上
    // 因为匿名箭头函数的 this 指向定义的时候
    return <input ref={input => this['input-' + index] = input} />;
  }

  render() {
    return <DataTable data={this.props.data} renderRow={this.renderRow} />
  }
}
```

- 命名冲突问题。直接挂载在`this`，可能出现相同字符串而导致命名冲突

```js
// 冲突
this.refs.myRef
<div ref="myRef"></div>
```
- 不可被深层次地传递并组合。而`callback ref`因为是函数更加灵活，比如在`React.cloneElement`的时候，可以将`ref`属性当作函数进行回调，让外部的自组件决定如何取值。

```js
React.cloneElement(children, {
  ref: child => {
    this.childRef = child;
    children.ref && children.ref(child);
  }
});
```

更多详情可见：[相关 issue](https://github.com/facebook/react/pull/8333#issuecomment-271648615) 以及 [React ref 的前世今生](https://zhuanlan.zhihu.com/p/40462264)

### callback ref 的注意事项

如果 callback ref 是`inline function`的形式（也就是`ref={() => { }}`），那在**更新阶段**中会被执行两次，一次被置为`null`，一次才是获取 DOM 元素。这是因为在每次渲染的时候，`inline function`的形式会生成新的实例。不过可以在类中使用`bound method`或类的成员方法，也就是使用形如`ref={this.boundRef}`这样的用法。需要注意的是无论`bound method`的形式还是`inline function`，在**组件卸载**的时候都会被以参数为`null`执行一次，组件挂载的时候以参数`DOM element`执行一次。

为什么`inline function`的形式在**更新阶段**会被执行两次，而`bound method`只有一次？

Dan 基于各种情况给出了 [解释](https://github.com/facebook/react/issues/9328#issuecomment-298438237) ，下面给出个人解读，更多详细内容见原解释。

- 为什么在卸载的时候需要执行一次`ref(null)`？因为`callback ref`的传递和组合，在深层次的传递中，可能有多个基于 ref 的副作用，如果卸载的时候不执行一次，基于 ref 的各层级根本无法得知，且可能会造成内存泄漏（因为一直引用卸载节点，无法被 GC）。而执行一次`ref(null)`是最简单有效的方式
- 在`ref`函数引用变更的时候，为什么在节点**更新前**要执行一次`ref(null)`？这是考虑到三目运算符的情况`onChange={cond ? prevListener : nextListener}`，这种情况为了第一种情况的原因，也要先执行`ref(null)`进行周知，再真正地调用`ref(dom)`
- 箭头函数在 re-render 的时候也会出现上述的「引用变更」情况，所以也执行一次`ref(null)`

而文章 [React ref 的前世今生](https://zhuanlan.zhihu.com/p/40462264) 也从源码的角度进行了解释。在`ref`变更的情况下，会先后调用`commitDetachRef`进行`ref(null)`，再调用`commitAttachRef`执行`ref(dom)`

![ref 构建流程](http://image.geekaholic.cn/20191029201003.png@0.8)

### react-redux 中 ref 的注意事项

`react-redux`从 V6 开始，增加了`{ forwardRef: true }`配置项，用于以后替换`{ withRef: true }`。而且区别在于之前的`{ withRef: true }`在获取的实例需要使用 HOC 组件实例`ref={(target) => this.instance = target}`的`this.instance.getWrappedInstance()`才可以获得被`connect`包裹的组件实例。而 V6 开始，借助`React.forwardRef()`可以直接地通过`ref={(target) => this.instance = target}`中的`this.instance`就可以获得！

---

参考：

* [Update nuances of ref callback calling by unel · Pull Request #8333 · facebook/react](https://github.com/facebook/react/pull/8333#issuecomment-271648615)
* [[BUG] ref function gets called twice on update (but not on first mount), first call with null value. · Issue #9328 · facebook/react](https://github.com/facebook/react/issues/9328)
* [React ref 的前世今生](https://zhuanlan.zhihu.com/p/40462264)
* [你想知道的关于 Refs 的知识都在这了 - 掘金](https://juejin.im/post/5db6506d6fb9a0207326a928#heading-8)
