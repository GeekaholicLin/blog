---
title: 模拟实现 JavaScript 中的私有变量
date: 2019-11-1 19:35:00
tags: [JavaScript, React, 读书笔记]
categories: [JavaScript]
---

### 命名约定

```js
class Shape {
  constructor(width, height) {
    this._width = width;
    this._height = height;
  }
  get area() {
    return this._width * this._height;
  }
}

const square = new Shape(10, 10);
console.log(square.area);    // 100
console.log(square._width);  // 10

```

优点：简单

缺点：只是一种约定，依然可以从外部获取所谓的私有变量

### WeakMap

```js
const map = new WeakMap();

// 创建一个在每个实例中存储私有变量的对象
const internal = obj => {
  if (!map.has(obj)) {
    map.set(obj, {});
  }
  return map.get(obj);
}

class Shape {
  constructor(width, height) {
    internal(this).width = width;
    internal(this).height = height;
  }
  get area() {
    return internal(this).width * internal(this).height;
  }
}

const square = new Shape(10, 10);
console.log(square.area);      // 100
console.log(map.get(square));  // { height: 100, width: 100 }

```

优点：可以有效防止属性遍历以及`JSON.stringify`时可见；WeakMap 的优势使得私有变量跟随实例，实例销毁，WeakMap 所在的对应属性也会被回收。

缺点：需要额外依赖 WeakMap

### Symbol

```js
const widthSymbol = Symbol('width');
const heightSymbol = Symbol('height');

class Shape {
  constructor(width, height) {
    this[widthSymbol] = width;
    this[heightSymbol] = height;
  }
  get area() {
    return this[widthSymbol] * this[heightSymbol];
  }
}

const square = new Shape(10, 10);
console.log(square.area);         // 100
console.log(square[widthSymbol]); // 10， Symbol 只要不暴露外部无法获取

```

优点：可以有效防止`JSON.stringify`时可见；

缺点：使用特定的遍历方法依然可以遍历出相关属性，比如`Reflect.ownKeys`等能获取 symbol 类型的变量；需要借助`Symbol`

### 闭包

```js
// 工厂模式
// 可以与 WeakMap 或者 Symbol 结合使用
function Shape() {
  // 私有变量集（包括私有方法）
  const this$ = {};

  class Shape {
    constructor(width, height) {
      this$.width = width;
      this$.height = height;
    }

    get area() {
      return this$.width * this$.height;
    }
  }

  const instance = new Shape(...arguments); // 实际上的实例
  // 设置实际实例的原型的原型为外部的 Shape，否则`square instanceof Shape`为 false
  Object.setPrototypeOf(Object.getPrototypeOf(instance), this);
  return instance;
}

const square = new Shape(10, 10);
console.log(square.area);             // 100
console.log(square.width);            // undefined
console.log(square instanceof Shape); // true
```

缺点：需要包裹多一层，封装实现复杂

### Proxy

```js
// 与方法一结合，进行增强
// 代理模式
class Shape {
  constructor(width, height) {
    this._width = width;
    this._height = height;
  }
  get area() {
    return this._width * this._height;
  }
}

const handler = {
  get: function(target, key) {
    if (key[0] === '_') {
      throw new Error('Attempt to access private property');
    } else if (key === 'toJSON') { // <= 增加对`JSON.stringify`的处理，过滤掉私有属性
      const obj = {};
      for (const key in target) {
        if (key[0] !== '_') {
          obj[key] = target[key];
        }
      }
      return () => obj;
    }
    return target[key];
  },
  set: function(target, key, value) {
    if (key[0] === '_') {
      throw new Error('Attempt to access private property');
    }
    target[key] = value;
  },
  getOwnPropertyDescriptor(target, key) { // <= 更改属性的描述符号为不可枚举
    const desc = Object.getOwnPropertyDescriptor(target, key);
    if (key[0] === '_') {
      desc.enumerable = false;
    }
    return desc;
  }
}

const square = new Proxy(new Shape(10, 10), handler);
console.log(square.area);             // 100
console.log(square instanceof Shape); // true
console.log(JSON.stringify(square));  // "{}"
for (const key in square) {           // No output
  console.log(key);
}
square._width = 200;                  // Error: Attempt to access private property

```

---

参考：
* [[译] JavaScript 中的私有变量 - 掘金](https://juejin.im/post/5a8e9b6d5188257a5f1ed826#heading-1)