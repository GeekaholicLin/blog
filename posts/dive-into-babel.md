### 大致过程

Babel 的转换 总共分为三个阶段：解析（parse）、转换（transform）以及生成（generate）。

![Babel 运行过程](http://image.geekaholic.cn/20191107114744.png@0.8)

- 解析：将代码通过 Babel 编译器`babylon`（基于 [acorn](https://github.com/acornjs/acorn)，后改名为`@babel/parser`）转换为 [ESTree](https://github.com/estree/estree) 规范（此规范为 SpiderMonkey 引擎输出的 JavaScript AST 规范） 的 AST 抽象语法树
- 转换：通过`babel-traverse`遍历访问 AST 节点，并对其进行相关操作（`babel-core`中的相关`transform`接口），进行转换生成新的 AST。这是最复杂的部分，也是 Babel 插件介入的部分。
- 生成：使用`babel-generator`以新的 AST 生成新的代码，同时还可以创建 source map

### 解析

而「解析」步骤又可以细分分为「词法分析」以及「语法分析」：

- 词法分析（Lexical Analysis），词法解析器（Tokenizer）在这个阶段将代码当成字符串，并转换为令牌（Tokens）。Tokens 一般为数字、标识符、操作符或者 JavaScript 中的符号（比如`{`、`}`、`(`、`)`等等）
- 语法分析（Syntactic Analysis），解析器（Parser）会根据 Tokens 中的信息，对各种从属关系进行考虑，把 Tokens 转换为 AST，这样方便于后续的操作。

> 简单来说语义分析是对语句（Statement）和表达式（Expression）识别，这是个递归过程，在解析中，babel 会在解析每个语句和表达式的过程中设置一个暂存器，用来暂存当前读取到的语法单元（stash），如果解析失败，就会返回之前的暂存点（rewind），再按照另一种方式进行解析，如果解析成功，则将暂存点销毁（commit），不断重复以上操作，直到最后生成对应的语法树。 -- [Babel 插件原理的理解与深入](https://github.com/frontend9/fe9-library/issues/154)

我们以下边代码为例子（`babylon-6`为解析器），进行解说。可以通过 [此链接](https://astexplorer.net/#/gist/f712105c9054c268d9bb8948430d4886/fe6aea8940cce220978a3715627321c4628a690c) 在线查看对应的结果。

```js
function square(n) {
  return n * n; // 返回结果
}
```

词法分析的结果如下：

![Tokens](http://image.geekaholic.cn/20191108142323.png@0.8)

而语法分析会将上边的 Tokens 转换为 AST，如下图：

![初始 AST](http://image.geekaholic.cn/20191108143154.png@0.8)

可以发现上述的代码被转换为树形结构，这就是 AST（抽象语法树）。AST 的每一层称为节点，而 AST 正是由单个或成百上千个节点构成。每个节点都拥有节点类型`type`，比如表示函数声明的`"FunctionDeclaration"`。除此之外还有节点类型对应的附加属性用于描述该节点，比如`"FunctionDeclaration"`类型的节点中有`"params"`属性，表示了接收的参数，而`"params"`属性每一个值又都是`"Identifier"`类型的节点。

AST 十分的复杂，且不同的解析器（`astexplorer`中可以选定解析器）解析的结果可能不同，所以记下所有的节点类型显然是不现实的，我们可以在需要的时候借助`astexplorer`进行分析即可。

### 遍历

开始之前，先大概说明一下「语句」、「表达式」以及「声明」的区别：

- 语句（Statement)：常见的判断语句、循环语句、异常处理语句等都属于语句
- 表达式（Expression）：一组代码的组合，会返回一个值。比如算术表达式、逻辑表达式、函数表达式等
- 声明（Declaration）：分为变量声明和函数声明

我们把上面的例子输出的 AST 简化成下边的对象：

```js
/*
function square(n) {
  return n * n;
}
*/

{
  type: "FunctionDeclaration",
  id: {
    type: "Identifier",
    name: "square"
  },
  params: [{
    type: "Identifier",
    name: "n"
  }],
  body: {
    type: "BlockStatement",
    body: [{
      type: "ReturnStatement",
      argument: {
        type: "BinaryExpression",
        operator: "*",
        left: {
          type: "Identifier",
          name: "n"
        },
        right: {
          type: "Identifier",
          name: "n"
        }
      }
    }]
  }
}
```

Babel 的遍历是**深度优先**，也就是依次遍历每一个属性及其子节点。

### 访问者（Visitors）

在遍历的时候，「进入」一个节点可以当作是在「访问」它。采用「访问者」这一概念是因为存在「访问者模式」-- 元素的执行算法可以随着访问者改变而改变。

> 首先我们拥有一个由许多对象构成的对象结构，这些对象的类都拥有一个 accept 方法用来接受访问者对象；访问者是一个接口，它拥有一个 visit 方法，这个方法对访问到的对象结构中不同类型的元素作出不同的反应；在对象结构的一次访问过程中，我们遍历整个对象结构，对每一个元素都实施 accept 方法，在每一个元素的 accept 方法中回调访问者的 visit 方法，（而回调访问者中拥有多个重载的 visit 方法，参数为具体的对象类型）从而使访问者得以处理对象结构的每一个元素。我们可以针对对象结构设计不同的实在的访问者类来完成不同的操作。 -- 维基百科

JavaScript 中没有方法重载，但可以利用对象的结构达到以上「随着访问者改变而改变」的目的（见下面例子中的 visitor 的结构）。

在 Babel 中，访问者用于 AST 遍历，它是一个定义了获取具体节点的方法的对象。而且在 Babel 中的访问者每一个具体类型可以定义「enter」和「exit」两个触发动作的方法，分别表示「访问该节点」和「结束访问该节点」，需要注意的是「exit」是访问**所有子节点结束**后退出而不仅仅是当前节点。可以看下面的代码例子：

```js
// 此访问者对象会在进入`Identifier`类型的时候调用
// enter 的 简写类型
const MyVisitor = {
  Identifier(path) {
    console.log("Called!");
  }
};
// 两个触发时机
const MyVisitor = {
  Identifier: {
    enter(path) {
      console.log("Entered!");
    },
    exit(path) {
      console.log("Exited!");
    }
  }
};
// 多个节点类型共用一套逻辑
const visitor = {
  "FunctionExpression|ArrowFunctionExpression"() {
    console.log("A function expression or a arrow function expression!");
  }
};
```

而在 Babel 中，多个插件的情况会怎么应用？在访问某个节点的时候，会根据 Plugin 定义的顺序或者 preset 定义的逆序，依次触发对应的访问者中的具体方法。逻辑数据结构如下：

```js
{
  Identifier: {
    enter: [plugin - xx, plugin - yy]; // 数组形式
  }
}
```

编写访问者对象是 Babel 插件编写中最重要的一部分。_编写的宗旨是不能破坏修改部分以外的其他代码，保证程序的正确性_，而编写过程中也有一些概念以及注意事项是需要我们了解的。

#### 路径（Paths）

访问者在遍历 AST 的时候调用对应方法，其中参数并不是具体的节点本身，而是一个路径对象。

> 路径是一个节点在树中的位置以及关于该节点各种信息的响应式表示

「响应式」指的是调用一个修改树的方法后，路径信息也会被更新。这是由 Babel 维护和管理的，可以使得节点操作简单。

路径是用来表示节点与节点直接的关联关系。 一个路径对象包含了很多元信息（当前节点信息，节点关联信息，作用域信息，上下文信息等），操作节点的相关方法（比如`traverse`、`remove`、`replaceWith`等）以及断言方法。

#### 状态（State）

需要注意的是，访问者的编写对相应**拥有子节点的节点**进行处理，要时时刻刻考虑访问者是否会**意外地**对其他地方进行修改。

```js
/*
将下列的 n 转换为 x
function square(n) {
  return n * n;
}
*/
let paramName;

const MyVisitor = {
  FunctionDeclaration(path) {
    const param = path.node.params[0];
    paramName = param.name;
    param.name = "x";
  },

  Identifier(path) {
    if (path.node.name === paramName) {
      path.node.name = "x";
    }
  }
};
```

以上访问者的编写错误在于，`Identifier`节点有可能是函数之外的，会污染函数之外的部分。正确的做法应该是遍历传递 `paramName`：

```js
const updateParamNameVisitor = {
  Identifier(path) {
    if (path.node.name === this.paramName) {
      path.node.name = "x";
    }
  }
};

const MyVisitor = {
  FunctionDeclaration(path) {
    const param = path.node.params[0];
    const paramName = param.name;
    param.name = "x";
    // 递归，并将 paramName 传下去，好处是避免影响修改以外的部分
    path.traverse(updateParamNameVisitor, { paramName });
  }
};

path.traverse(MyVisitor);
```

#### 作用域（Scopes）

编写访问者对象的时候为了保证不对**处理部分以外**的代码做修改，除了需要考虑对代码外其他东西的处理，还要考虑对作用域的处理，比如新增引用要确保新增的引用名字不和已有的引用冲突。因为这样的处理 Babel 是检测不出来异常的，只能靠开发者自身进行规范。

变量、函数、类、参数等都属于在声明时所在的作用域，在当前作用域或作用域以下的作用域使用这些标识符时，称之为引用（reference）。所有引用属于特定的作用域。

而绑定（bindings）指的是引用（reference）与隶属作用域（scope）的关系。（原文为：「References all belong to a particular scope; this relationship is known as a binding.」）

作用域对象包含了当前作用域信息、父级作用域信息、当前作用域的所有 bindings 以及对作用域的操作方法。一个作用域（scope）在 Babel 中表示为：

```js
/*
// 例子
import { parse } from '@babel/parser'
import traverse from '@babel/traverse'

const ast = parse(`function scopeOnce() {
  var ref = "This is a binding";

  ref; // This is a reference to a binding

  function scopeTwo() {
    ref; // This is a reference to a binding from a lower scope
  }
}`)
traverse(ast, {
  FunctionDeclaration(path) {
    console.log(path.scope)
  }
})
*/
{
  path: path, // 路径对象
  block: path.node, // 所属作用域所在节点
  parentBlock: path.parent, // 父级作用域所在节点
  parent: parentScope, // 父级作用域
  bindings: { // 当前作用域下的引用
     ref: { // 标识符名称，值为 Binding 对象
        identifier: [Object],
        scope: [Circular], // 所在的作用域（就是 bindings 所在的这个对象），显示 Circular 是因为循环引用了
        path: [Object],
        kind: 'var', // 枚举值，"var" | "let" | "const" | "module"
        constantViolations: [],
        constant: true, // 是否是常量（指赋值后不修改，只读取，而不是 const）
        referencePaths: [Array], // 所有引用的路径对象
        referenced: true, // 是否被引用
        references: 2, // 被引用次数
        hasDeoptedValue: false,
        hasValue: false,
        value: null
     }
    }
}
```

作用域的相关操作十分麻烦，不过实际插件编写中很少使用。作用域操作最典型的场景是代码压缩，代码压缩会对变量名、函数名等进行压缩。因为涉及修改变量名，防止污染其他作用域的变量。

下面给出一个来自 [文章](https://juejin.im/post/5d94bfbf5188256db95589be) 的例子：

```js
import { parse } from "@babel/parser";
import traverse from "@babel/traverse";
import generate from "@babel/generator";

// 要求：对 add 函数中的 foo 重命名

const ast = parse(`const a = 1, b = 2
function add(foo, bar) {
  console.log(a, b)
  return () => {
    const a = '1' // 新增了一个变量声明
    return a + (foo + bar)
  }
}`);
traverse(ast, {
  FunctionDeclaration(path) {
    const firstParam = path.get("params.0"); // 获取第一个参数
    // 需要函数为 add 且第一个参数为 foo
    if (
      !firstParam ||
      firstParam.node.name !== "foo" ||
      firstParam.parent.id.name !== "add"
    ) {
      return;
    }
    let i = path.scope.generateUidIdentifier("_"); // 1. 生成唯一标识符对象
    const currentBinding = path.scope.getBinding(firstParam.node.name); // 2. 获取引用的绑定对象
    currentBinding.referencePaths.forEach(p => p.replaceWith(i)); // 3. 对引用所有引用路径进行修改
    firstParam.replaceWith(i); // 4. 第一个参数本身进行修改
    // 5～6 的效果如同 1～4
    // let i = path.scope.generateUid('_') // 5. 生成唯一标识符字符串
    // path.scope.rename(firstParam.node.name, i) // 6. 重命名
  }
});
console.log(generate(ast).code);
```

### 插件开发常用到的工具

- `@babel/core`: 读取并处理配置，加载插件；解析代码为 AST，遍历 AST；遍历 AST 的时候，应用插件对 AST 进行操作；根据新的 AST 生成新的代码（包括 source map）
- `@babel/parser`: 将 JS、Flow 或 JSX 等语法的代码转换为 AST，基于`acorn`
- `@babel/traverse`：对 AST 进行遍历，方便**后续**对节点进行替换、删除或添加等操作（**操作是 Babel 插件的职责**）
- `@babel/types`: AST 的类型包，用于节点构造、节点断言和节点转换
- `@babel/template`：模版引擎，提供句法占位符以及标识符占位符，可以将大量的代码字符串转换为 AST，方便**后续**对 AST 进行操作
- `@babel/helper-*`：辅助函数，用于辅助插件开发

#### @babel/types 的使用

`@babel/types`对每一个节点的类型都有一个定义，通过这个节点类型的定义可以知道其节点有哪些属性又在哪些位置（比如二元表达式分为操作符、左表达式和右表达式三个属性，左表达式在运算符左边，右表达式同理）、节点的值是否合法、如何构建该节点、如何遍历节点以及节点的别名。

我们先来看一下一个节点类型的定义，然后根据类型定义推导出其他信息。

```js
defineType("BinaryExpression", {
  builder: ["operator", "left", "right"],
  fields: {
    operator: {
      validate: assertValueType("string")
    },
    left: {
      validate: assertNodeType("Expression")
    },
    right: {
      validate: assertNodeType("Expression")
    }
  },
  visitor: ["left", "right"],
  aliases: ["Binary", "Expression"]
});
```

上面的代码是一个二元表达式`BinaryExpression`的定义。

这个定义中`builder`构造器属性指示了二元表达式的组成，再结合各个部分的`validate`属性，可以推导出二元表达式节点的构造方式 -- `t.binaryExpression("*", t.identifier("a"), t.identifier("b"))`，先是一个字符串类型的运算符，然后是表达式类型的左表达式和右表达式。

而各个部分的`validate`属性就是`validator`验证器了，其用于表明组成当前节点的子节点的类型以及验证方式。当代码中组成节点的子节点类型，与定义对应的验证器不符，**可以选择**返回布尔值或者抛出异常。

```js
t.isBinaryExpression(maybeBinaryExpressionNode); // types.isxxxx() 返回布尔值
t.isBinaryExpression(maybeBinaryExpressionNode, { operator: "*" }); // 同上，且限制了相关子节点的精确匹配值
t.assertBinaryExpression(maybeBinaryExpressionNode); // types.assertxxxx() 如果不通过抛出异常
t.assertBinaryExpression(maybeBinaryExpressionNode, { operator: "*" }); // 同上，且限制相关子节点的精确匹配值
```

#### @babel/template 的使用

```js
// @babel/template 的使用
import template from "@babel/template";
import generate from "@babel/generator";
import * as t from "@babel/types";

const buildRequire = template(`
  var IMPORT_NAME = require(SOURCE);
`);

const ast = buildRequire({
  IMPORT_NAME: t.identifier("myModuleName"),
  SOURCE: t.stringLiteral("./my-module/index.js")
});
console.log(generate(ast).code); // 'var myModuleName = require("./my-module/index.js");'
```

---

参考：

- [jamiebuilds/the-super-tiny-compiler: Possibly the smallest compiler ever](https://github.com/jamiebuilds/the-super-tiny-compiler)
- [babel-handbook/plugin-handbook.md at master · jamiebuilds/babel-handbook](https://github.com/jamiebuilds/babel-handbook/blob/master/translations/zh-Hans/plugin-handbook.md)
- [Babel 插件原理的理解与深入](https://github.com/frontend9/fe9-library/issues/154)
- [深入浅出 Babel 上篇：架构和原理 + 实战](https://juejin.im/post/5d94bfbf5188256db95589be)
