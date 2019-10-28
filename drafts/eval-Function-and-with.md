### eval

`eval`接收一个字符串作为参数，并调用 JavaScript 解析器进行执行。而如果参数不是字符串，则原样返回。

_如果通过引用来间接调用`eval`，则工作在全局作用域_。这里多说一下，`setTimeout`与`setInterval`的第一个参数为字符串的时候，也是工作在全局作用域。

```js
// eval 的作用域
function test() {
  var x = 2, y = 4;
  console.log(eval('x + y'));  // 直接调用，使用本地作用域，结果是 6
  var geval = eval; // 等价于在全局作用域调用
  console.log(geval('x + y')); // 间接调用，使用全局作用域，throws ReferenceError 因为`x`未定义
  (0, eval)('x + y'); // 另一个间接调用的例子
​}
// 字符串作为参数时的 setTimeout
(function(){
  var fakeClick = function(){
    console.log('fakeClick!')
  };
  setTimeout('fakeClick',1000);
})() // Uncaught ReferenceError: fakeClick is not defined
```

`eval` 中函数表达式作为字符串被定义需要用括号包裹作为前缀和后缀，否则会被执行，返回的是函数的返回值。`eval`中的对象为了防止被解析为语句块，也需要括号。可以把`eval('(' + data_from_the_wire + ')')`看作是`eval('return ' + data_from_the_wire + ';')`。

```js
// eval 执行的表达式与语句
var fctStr1 = 'function a() {}'
var fctStr2 = '(function a() {})'
var fct1 = eval(fctStr1)  // 返回 undefined
var fct2 = eval(fctStr2)  // 返回一个函数
eval("{ foo: 123 }") // 返回 '123'
eval('({ foo: 123 })') // 返回 { foo: 123 }
```

另外，在 ES2019 之前，行分隔符`\u2028`与段分隔符`\u2029`在`JSON.parse`中是合法的，但在 JavaScript 的字符串字面量中是非法字符。所以`eval`无法识别会抛出异常。

```js
var data = '{"name":"\u2028"}';
eval('('+data+')');//ES2019 之前会出错，Invalid or unexpected token
JSON.parse(data);//正常解析
````

所以用`eval`模拟`JSON.parse`需要进行一定的处理。

```js
function parseJSON(s) {
    if ('JSON' in window) return JSON.parse(s);
    return eval('('+s.replace(/\u2028/g, '\\u2028').replace(/\u2029/g, '\\u2029')+')');
}
```

永远也不要使用`eval`，它是一个危险的函数，可能被运行恶意的代码。且`eval`的过于灵活不可预测，会使得 JS 解释器难以优化。一般推荐使用`Function`进行替换。

### Function

`Function()`与`new Function()`的效果是一样的。语法为`new Function ([arg1[, arg2[, ...argN]],] functionBody)`。使用 Function 构造器生成的函数，并不会在创建它们的上下文中创建闭包；它们一般在**全局作用域**中被创建。

```js
const adder = new Function("a", "b", "return a + b")
adder(2, 6)

// 全局作用域
var n = 1;
function f(){
    var n = 2;
    var e = new Function("return n;");
    return e;
}
console.log (f()()); //1
```

### with

JavaScript 查找某个未使用命名空间的变量时，会通过作用域链来查找。而`with`会将某个对象添加为花括号内变量最近的一个作用域，意味着最先查找。这可以减少我们的代码量，使`with`可以减少不必要的路径解析。但由此也带来了代码混乱、编译器无法优化等弊端。而且如今减少代码量可以使用解构赋值或临时变量来解决。所以不推荐使用`with`，严格模式下也是不允许使用`with`的。

```js
// 语义不明
function f(x, o) {
  with (o){
   console.log('with: ', x) // 'with: ox'
  }
  console.log('f: ', x) // 'f: x'
}
f('x', { x: 'ox' })
// 语义不明
function f(obj, length) {
    with (obj) {
        console.log(length) // 3
    }
}
f([1,2,3], 'length')
```
