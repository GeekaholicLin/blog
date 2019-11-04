```js
// 题目来自 [研究 js 的块级作用域中的变量声明和函数声明 - 掘金](https://juejin.im/post/5d90ae9ef265da5b646480a0)
{
  function a(){}
  a = 50
}
console.log(a)
{
  b = 50
  function b(){}
}
console.log(b)
```

看上边的输出结果发现很有意思，与认知中的有出入，搜索了一下，发现 MDN 中就有说明，先给出相关的 MDN[链接](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Functions#Block-level_functions)

函数声明很容易被转换成函数表达式，只要满足以下任一条件：

1. 函数声明作为表达式的一部分
2. 函数声明不在全局的顶级或不在函数体的顶级进行声明（也就是需要 non-nested statement 才不会被转换，具体形式可以见上边给出的 MDN 链接）

MDN 把图中的那种函数声明叫「Block-level functions」，对应的规范在 [这里](https://tc39.es/ecma262/#sec-block-level-function-declarations-web-legacy-compatibility-semantics)。而且这里还有一个兼容性的问题：

```plain
// 来自上述 stackoverflow 中的回答，web-compat 表示实际浏览器的处理，pure 目测是表示 ES 中的规范或者说建议的处理（原回答没有说，个人猜测）

                 |      web-compat               pure
-----------------+---------------------------------------------
strict mode ES6  |  block hoisting            block hoisting
sloppy mode ES6  |  it's complicated          block hoisting
strict mode ES5  |  undefined behavior        SyntaxError
sloppy mode ES5  |  undefined behavior        SyntaxError
```

严格模式下，「Block-level functions」在 ES6 之前虽然规范中没有定义，但是浏览器实现一般是会报错；ES6 开始其作用域会被限制在声明所在的块，表现形式跟 var 类似，**变量声明**提升至所在块级作用域顶部，而只有在执行到该条语句的时候才会进行相关赋值。

而非严格模式下，「Block-level functions」的行为在 ES6 之前根据浏览器的不同而不同，从 [这个链接](http://w3help.org/zh-cn/causes/SJ9002) 中可以获得一些信息；而从 ES6 开始，非严格模式的处理就显得复杂了（**很大可能**也是根据浏览器的不同而不同，未测试。_以下以 Chrome 最新版为例_），伪代码如下：

```js
// 所写的代码
function enclosing() {
    // 块级作用域前执行的代码
    {
         // Block-level function 声明前执行的代码
         function compat() {  }
         // Block-level function 声明后执行的代码
    }
    // 块级作用域后后执行的代码
}

// 逻辑转换后的代码
function enclosing() {
    var compat₀ = undefined; // function-scoped
    // 块级作用域前执行的代码
    {
         let compat₁ = function compat() { }; // block-scoped
         // Block-level function 声明前执行的代码
         compat₀ = compat₁
         // Block-level function 声明后执行的代码
    }
    // 块级作用域后执行的代码
}
```

其中`compat₀`与`compat₁`都是名为`compat`的变量，只不过为了表示清楚，用了不同的变量名来表示。

从转换后可以看出，非严格模式下的 ES6 有两个变量提升。一个是在块级外面的函数作用域顶部，一个是在块级作用域的顶部。

现在对开头的题目进行逻辑转换：

```js
// 这里全局作用域顶部有 a 的声明，相当于 var a = undefined
{
  // 这里块级作用域顶部有 a 的声明与赋值，相当于 let a = function a(){}
  function a(){} // 执行到这里这里相当于赋值给全局作用域的 a
  a = 50 // 这里的 a 查找作用域的时候，发现块级作用域有 a，所以赋值块级作用域的 a
}
console.log(a) // 打印：function a(){} 。这里打印的是全局作用域的 a
// 这里全局作用域顶部有 b 的声明，相当于 var b = undefined
{
  // 这里块级作用域顶部有 a 的声明与赋值，相当于 let b = function b(){}
  b = 50 // 赋值块级作用域 b
  function b(){} // b 已经被覆盖为 50，所以外部全局作用域也为 50
}
console.log(b) // 打印：50
```

我们给出一个更加详细的打印代码：

```js
console.log('a:1 =>', window.a, a) // undefined, undefined
{
  console.log('a:2 =>', window.a, a) // undefined, function a(){}
  function a(){}
  console.log('a:3 =>', window.a, a) // function a(){}, function a(){}
  a = 50
  console.log('a:4 =>', window.a, a) // function a(){}, 50
}
console.log('a:5 =>', window.a, a) // function a(){}, function a(){}。注意这里的 a 和 window.a 都是全局作用域的，因为已经离开了块级作用域

console.log('b:1 =>', window.b, b) // undefined, undefined
{
  console.log('b:2 =>', window.b, b) // undefined, function b(){}
  b = 50 // 覆盖局部变量 b
  console.log('b:3 =>', window.b, b) // undefined, 50
  function b(){}
  console.log('b:4 =>', window.b, b) // 50, 50。这里是因为上一句中局部变量赋值给全局变量，而局部变量在`b=50`时被覆盖，从函数转变为 50
}
console.log('b:5 =>', window.b, b) // 50, 50。打印的都是全局的
```
