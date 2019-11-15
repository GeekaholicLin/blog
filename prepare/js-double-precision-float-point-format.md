在 JavaScript 中，所有的 Number 类型的数字都是以双精度浮点数（浮点数是指小数点飘忽不定，由阶码决定）的方式存储。这也带来了一些诡异的问题，比如`0.1 + 0.2 !== 0.3`。

本文主要由「知识储备」和「问题答疑」两个大章节组成，尝试地去解答以下问题：

1. 为什么`0.1 + 0.2 !== 0.3`？
2. 如何用`Math.pow()`去模拟`Number.MIN_VALUE`等 Number 的常量？
3. `toPrecision()`、`toFixed()`以及`Math.round()`的区别是什么？
4. 为什么`(1.33500000000000001).toFixed(2) === '1.33'`而不是`'1.34'`？
5. 为什么`Number.MAX_SAFE_INTEGER + 1 === Number.MAX_SAFE_INTEGER + 2`？
6. 为什么`0.100000000000000002 === 0.100000000000000010`？
7. 大于`Number.MAX_VALUE`的数表示无穷大，那为什么`Number.POSITIVE_INFINITY !== Number.MAX_VALUE + 1`？

特别地，问题 5-7 归结为「相等判定与数量级」问题。

## 知识储备

### 存储结构

![双精度浮点数的存储结构]](http://image.geekaholic.cn/20191114154418.png@0.8)

- 符号位 S：0 代表正数，1 代表负数
- 指数位 E：中间的 11 位存储指数，用来表示次方数
- 尾数位 M：最后的 52 位是尾数，超出部分使用下面章节的 [舍入模式](#%e8%88%8d%e5%85%a5%e6%a8%a1%e5%bc%8f)（虽然用 52 位来存储，但可表示 53 位数字）

用公式表示即为：

![公式表示 1](http://image.geekaholic.cn/20191114154505.png@0.8)

对于目前的指数位 E 而言，其能表示的范围为`[0, Math.pow(2, 11) - 1]`，也就是`[0, 2047]`。但是在科学计数法中指数是可以为负数的，所以需要减去一个偏移量`1023`（至于偏移量为什么是`1023`，暂且当作是规定），使得科学计数法中的指数能够表示的数的范围为`[-1023, 1024]`。且在二进制的科学计数法中，尾数的小数点前面必然是数字`1`，为了能够让尾数表示更多位以最大限度地提高精度，数字`1`可以省略。不省略的情况下尾数位 M 部分是 1 位整数 + 51 位小数，现如今省略掉整数则 52 位都可以表示尾数的小数部分，加上省略掉的整数`1`则可表示 53 位数字。

综上所述，公式变成了：

![公式表示 2](http://image.geekaholic.cn/20191114160844.png@0.8)

### 规约形式与非规约形式

前面讲到的「存储结构」其实指的是规约形式（normal）的浮点数存储，它的表现形式如同公式所言，尾数位 M 表示的是尾数的小数部分，整数部分默认为`1`。

指数部分 E 虽然取值范围为`[-1023, 1024]`，但是全 0 和全 1 用于其他用途，所以表示规约形式（normal）双精度浮点数的时候，E 的有效范围为`[-1022, 1023]`。

而非规约形式（subnormal）的浮点数指的是尾数的整数部分不再是默认`1`，而是`0`。且阶码 E 为全 0，尾数部分不为全 0。这种形式称之为非规约形式。根据标准规定，「非规约形式的浮点数的指数偏移值比规约形式的浮点数的指数偏移值大 1」，所以在双精度浮点数的时候，指数部分固定为`-1023 + 1 = -1022`，用公式表示如下：`V = (-1)^S * (2 ^ -1022) * (0 + M)`

为什么还需要非规约形式的浮点数？它是用来填补**绝对值**意义下最小规格数与零的距离之间的数。

![非规约](http://image.geekaholic.cn/20191115112221.png@0.8)

这里借助 [维基百科](https://en.wikipedia.org/wiki/Denormal_number) 中的一个图片（见上图）来进行说明。

图中以正数部分为例（负数部分同理），红色表示在规约形式的表示系统里能表达的数，蓝色是非规约形式所能表达的数，它填补了下溢区间（underflow gap，指的是规约形式所能表达正负最小值的区间），能够表示非常接近 0 的实数。它的意义在于引入了渐进下溢（gradual underflow）而不是突然下溢（突然下溢的结果是会将对应的数改为零）。

规约形式的最小正浮点数为`(2^-1022) * 1.0`（E 不能为全 0），而比它最近的数为`(2^-1022) * (1 + (2^-52))`，差值为`2^(-1022-52)=2^-1074`，这已经超出了规约形式能表示的最小值，如果不引入非规约形式的表示，那么在计算两个十分相近的规约形式的浮点数时，很可能会导致突然下溢而结果为 0（flush to zero）。

### 双精度浮点数的所有情况

结合规约形式、非规约形式以及一些特殊值，得到一张这样的表格，用于概括双精度浮点数数值的表示情况：

![表示情况](http://image.geekaholic.cn/20191115152629.png@0.8)

对于 NaN，则 E 部分为全 1 且 M 部分不为 0，从 [维基百科](https://en.wikipedia.org/wiki/Double-precision_floating-point_format) 中得出三种不同的 NaN 表示。

```plain
0 11111111111 0000000000000000000000000000000000000000000000000000 // 正无穷大，Number.POSITIVE_INFINITY
1 11111111111 0000000000000000000000000000000000000000000000000000 // 负无穷大，Number.NEGATIVE_INFINITY
0 00000000000 0000000000000000000000000000000000000000000000000000 // +0
1 00000000000 0000000000000000000000000000000000000000000000000000 // -0
0 11111111111 1111111111111111111111111111111111111111111111111111 // NaN
0 11111111111 0000000000000000000000000000000000000000000000000001 // sNaN
0 11111111111 1000000000000000000000000000000000000000000000000001 // qNaN
```

### 舍入模式

二进制浮点数的舍入不是平时使用的「四舍五入」（对应到二进制就是「零舍一进」），而是「Round to nearest, ties to even」，叫「银行家舍入」，实质是「四舍六入五取偶」或叫「四舍六入五留双」。

处理规则就是：四舍六入五考虑，五后非零就进一，五后为零看奇偶，五前为偶应舍去，五前为奇要进一。

「ties to even」或者说「五取偶」的意思就是「Round half to even，看前面的数奇偶性，凑成偶数」，对应了处理规则中的「五前为偶应舍去，五前为奇要进」。

这个「五」在十进制中很好理解，就是`5`，那对应到二进制中要怎么看这个「四舍六入五取偶」呢？

借助 [IEEE754 规范的舍入方案怎么理解呢？ - Hael C 的回答](https://www.zhihu.com/question/68131179/answer/260937324) 中的课件和解释，来看一下在二进制中的舍入是怎样子的。

![二进制浮点数的舍入](http://image.geekaholic.cn/20191115140946.png@0.8)

使用了`GBS`三位数来分别对应保留位（Guard bit），近似位（Round bit）以及粘滞位（Sticky bit）。其中保留位即近似后的最低位，近似位即保留位的后一位，而粘滞位是近似位之后所有数值的 OR 运算（也就是只要是近似位之后存在一个`1`，那么粘滞位就是`1`）。

判断是否大于一半，主要是判断`BS`与`10`的关系（0.5，转换为二进制就是 0.10，取小数点后的数即`10`），`G`主要是用于在`BS`为`10`的时候取偶判断。下面`GBS`中的`x`表示`0`或`1`皆可。

- `GBS`为`x11`：`11 > 10`，那么说明是大于一半，需要进 1
- `GBS`为`x10`：这种情况就是前面说的「五」，这时候就看`x`的取值了。如果是偶数`0`，则舍弃；如果是奇数`1`，则需要进 1。
- `GBS`为`x01`：说明是小于一半的，舍弃
- `GBS`为`x00`：说明是小于一半，舍弃

可以看课件图片中的例子体会一下「四舍六入五取偶」在二进制中的使用。

### 换算方法

本小节以十进制中的`173.8125`为例子进行换算。

#### 手动换算

原则：整数部分使用「除 2 取余，逆序排列」，小数部分使用「乘 2 取整，顺序排列」

换算过程如下：

```plain
// 先转换整数 173，结果为 10101101
173 / 2 = 86...1
86 / 2 = 43...0
43 / 2 = 21...1
21 / 2 = 10...1
10 / 2 = 5...0
5 / 2 = 2...1
2 / 2 = 1...0
1 / 2 = 0...1

// 再转换小数 0.8125，结果为 1101
0.8125 * 2 = 1...0.625
0.625 * 2 = 1...0.25
0.25 * 2 = 0...0.5
0.5 * 2 = 1...0

// 整合整数和小数，结果为 10101101.1101
```

十进制数`173.8125`转换为二进制为`10101101.1101`，根据双精度浮点数的表达公式，则`S = 0,E = 1030, M = 0.01011011101`

#### 自动换算

直接借助 [工具](http://www.binaryconvert.com/convert_double.html)，进行换算。

![换算结果](http://image.geekaholic.cn/20191114165534.png@0.8)

将上图和手动换算的结果对应起来，首先`S`为 0，而`E = 1030 = (1030).toString(2) = 10000000110`，对于`M`只取小数点后的部分`01011011101`，正好对应图片中的表示。

### Number 中的常量

JavaScript 内置对象 `Number` 中有一些属性，用于表示常见的常量。一一来看这些常量和双精度浮点数的表示方法中有什么关联。

#### Number.EPSILON

表示 1 和`Number`可表示大于 1 的最小浮点数的差值，大小为`2^-52`，可用于比较的容差处理，判断结果是否足够精确以致于可以当成相等，比如`Math.abs(0.1+0.2-0.3) < Number.EPSILON`。

那`Number.EPSILON`的值怎么来的？

先来看「`Number`可表示的大于 1 的最小浮点数」用双精度浮点数的存储结构怎么表示？

```plain
0 01111111111 0000000000000000000000000000000000000000000000000001
```

符号位为 0，指数部分为`parseInt(0b01111111111, 10) - 127 = 0`，尾数位 M 为`2^-52`（当然这里还原后还得加上尾数的整数部分`1`）。所以，「大于 1 的最小浮点数」为`(-1)^0 * 2^0 * (1 + 2^-52)`，得出`Number.EPSILON`为`2^-52`。

#### Number.MAX_SAFE_INTEGER 与 Number.MIN_SAFE_INTEGER

`Number.MAX_SAFE_INTEGER` 表示 Number 类型中最大的安全整数，大小为`2^53-1`，转换为十进制则为`9007199254740991`。`Number.MIN_SAFE_INTEGER`表示最小安全整数，由`Number.MAX_SAFE_INTEGER`可得，符号位取反即可 -- 值为`-(2^53 - 1)`，转换为十进制则为`-9007199254740991`

那什么叫「安全」？`2^53-1`这个值怎么来的？

「安全」指的是，在浮点数存储机制中，能够与实数建立一对一映射。_这样用于比较数值的时候才是安全可靠的。_，比如`Number.MAX_SAFE_INTEGER + 1 === Number.MAX_SAFE_INTEGER + 2`结果为`true`，但从数学意义上而言是错误的。

如果要一一映射，那必须得利用尾数的有限的二进制位，也就是尾数的小数位数必须小于等于`52`。尾数 M 小数位数如果为`53`位，那么第`53`位就会因为存在自动进 1 舍 0，与其下一个数的表示（原有基础上加 1 进 1 而不是自动进 1）冲突导致不安全。

比如`2^53`和`2^53+1`，用双精度浮点数的存储结构怎么表示？

要使得公式`V = (-1)^S * (2^(E - 1023)) * (1 + M)`的结果为整数，那么：

- 对于`2^53`，符号位 S 为 `0`，指数位 E 为`E = 1023+53 = 1076 = 0b10000110100`，尾数位 M 都为 `0.0{52}`
- 对于`2^53+1`，符号位 S 为`0`，指数位 E 为`E = 1023+53 = 1076 = 0b10000110100`，实际尾数位 M 为`0.0{52}1`，也就是`M = 2^-53`。但是这里根据「四舍六入五取偶」规则，舍弃末尾的 `1`，导致`2^53+1`的表示与`2^53`相同，这也是为什么`Number.MAX_SAFE_INTEGER+1 === Number.MAX_SAFE_INTEGER+2`的结果为`true`。

```plain
0 10000110100 0000000000000000000000000000000000000000000000000000
0 10000110100 0000000000000000000000000000000000000000000000000000(1) // 根据舍入处理规则，会自动舍弃第 53 位

```

所以，为了「安全」地表示整数，整数转换为二进制后的 M 尾数不能超过 52。则「最大的安全整数」应该是`2^53 - 1`（临界点`53 位`都为 0，后退 1 步）。

判断一个数是否是安全整数，可以使用`Number.isSafeInteger()`：

```js
Number.isInteger = Number.isInteger || function(value) {
  return typeof value === 'number' &&
    isFinite(value) &&
    Math.floor(value) === value;
};
Number.isSafeInteger = Number.isSafeInteger || function (value) {
   return Number.isInteger(value) && Math.abs(value) <= Number.MAX_SAFE_INTEGER;
};
```

#### Number.MAX_VALUE

与`Number.MAX_SAFE_INTEGER`不同，`Number.MAX_VALUE`是表示双精度浮点数能表示最大的值，**接近于**`2^1024`。如果比`Number.MAX_VALUE`还大，那将被表示成`Infinity`。

那**接近于**`2^1024`这个值怎么来的？

指数部分 E 全 0 和全 1 是特殊用途，那 E 最大为`2046`，对应的指数部分为`1023`。而尾数 M 部分当然是全为 1 的时候最大。二进制表示如下：

```plain
0 11111111110 1111111111111111111111111111111111111111111111111111
```

假设加上一个`2^-52`，那么尾数的小数部分溢出为`2^0`，所以尾数的小数部分应该为`2^0 - 2^-52`。根据公式则得出`2^1023 * (1+(2^0 - 2^-52)) = 2^1024 - 2^971`，**接近于**`2^1024`。

那为什么不直接说`Number.MAX_VALUE`值为`2^1024 - 2^971`，而要说**接近于**`2^1024`？这是因为`2^1024`表示`Infinity`，而`Infinity`减去一个数依然为`Infinity`。

#### Number.MIN_VALUE

表示 JavaScript 能表示的最小正数，无限地接近 0，等于`2^-1074`。注意，它并不是`Number.MAX_VALUE`的符号位取反得来的值。

那这个数`2^-1074`又是怎么来的呢？

前面讲到双精度浮点数的表示形式有两种：规约与非规约。要想无限地接近 0，那只能使用非规约的形式。那现在以「非规格化」的思路来思考`Number.MIN_VALUE`的取值。

也就是`S = 0, E = 0, M 的第 52 位为 1`，存储结构表示如下：

```plain
0 00000000000 0000000000000000000000000000000000000000000000000001
```

根据非规约形式的公式`V = (-1)^S * (2 ^ -1022) * (0 + M)`，结果就是`V = 2^(-1022) * (0 + (2 ^ -52)) = 2^-1074`

#### Number.POSITIVE_INFINITY 和 Number.NEGATIVE_INFINITY

在讲`Number.MAX_VALUE`的时候讲到，如果一个值比`Number.MAX_VALUE`还大，那将被表示成正无穷`Infinity`（也就是`Number.POSITIVE_INFINITY`）。

前面得知`Number.MAX_VALUE`的存储结构表示为：

```plain
0 11111111110 1111111111111111111111111111111111111111111111111111
```

而在双精度浮点数中，`Number.POSITIVE_INFINITY`的存储结构表示如下：

```plain
0 11111111111 0000000000000000000000000000000000000000000000000000
```

从直观上看就是，`Number.POSITIVE_INFINITY = Number.MAX_VALUE + 1`，但是实际上`Number.POSITIVE_INFINITY !== Number.MAX_VALUE + 1`（为什么？后面章节会讲）。`Number.MAX_VALUE`是一个无限接近于`2^1024`的数，而`Math.pow(2, 1024) === Infinity`

`Number.NEGATIVE_INFINITY`则是`Number.POSITIVE_INFINITY`的符号位取反，值为`- Infinity`，负无穷。

## 问题答疑

### 为什么`0.1 + 0.2 !== 0.3`

在计算浮点数的加减法时，一般运算步骤为：「转换」-> 「对阶」 -> 「尾数求和」-> 「规格化」 -> 「舍入」。以`0.1+0.2`为例子，走一遍流程。

#### 转换

先将十进制的`0.1`和`0.2`分别采用「乘 2 取整，顺序排列」的方法，手动转换为二进制。

```plain
// 0.1 的换算
0.1 * 2 = 0...0.2
0.2 * 2 = 0...0.4 // 循环点 B
0.4 * 2 = 0...0.8
0.8 * 2 = 1...0.6
0.6 * 2 = 1...0.2 // 循环点 A
...

// 0.2 的换算
0.2 * 2 = 0...0.4
0.4 * 2 = 0...0.8 // 循环点 B
0.8 * 2 = 1...0.6
0.6 * 2 = 1...0.2
0.2 * 2 = 0...0.4 // 循环点 A
...

```

再将两者转换为双精度浮点数的表示形式：

```plain
// 暂且保留至第 53 位，而第 54 位为粘滞位，`()`为保留位，`[]`为粘滞位
0.1 = (-1)^0 * (2^-4) * (1.(1001){12}1001(1)[1])
0.2 = (-1)^0 * (2^-3) * (1.(1001){12}1001(1)[1])
// 两者 GBS 都为 111，根据舍入原则得出
0.1 = (-1)^0 * (2^-4) * (1.(1001){12}1010)
0.2 = (-1)^0 * (2^-3) * (1.(1001){12}1010)
```

#### 对阶

发现运算的两个数的阶码不一样，需要进行对阶，使得两个数的阶码相同。对阶操作中，应该是小阶码加上差值的绝对值，使得与大阶码相同，同时将小阶码的尾数右移使得本身的值不变。而尾数右移又**可能**会发生舍入，使得精确度损失。

所以变成了：

```plain
0.1 = (-1)^0 * (2^-3) * (0.1(1001){12}101(0)[0]) // `()`为保留位，`[]`为粘滞位
0.2 = (-1)^0 * (2^-3) * (1.(1001){12}1010)
```

#### 尾数求和

```plain
   0.1100110011001100110011001100110011001100110011001101(0)
+  1.1001100110011001100110011001100110011001100110011010
= 10.0110011001100110011001100110011001100110011001100111(0)
```

#### 规格化

尾数为`10.xxxx`，整数部分不为`1`，应该修改阶码使得小数点移动，变成规约形式的浮点数（normal number）。

所以阶码部分加 1，变成`2^(-3+1) = 2^-2`，而尾数部分的转换为：

```plain

10.0110011001100110011001100110011001100110011001100111(0)
=>
1.0011001100110011001100110011001100110011001100110011(10)
```

#### 舍入

`1.0011001100110011001100110011001100110011001100110011(10)`中的`GBS`为`110`，根据「四舍六入五取偶」的舍入策略，应该进 1，所以尾数部分为`1.0011001100110011001100110011001100110011001100110100`，求得的结果为`0.30000000000000004`

#### 结论

`0.1`和`0.2`在从十进制转换为二进制的时候因为出现循环，而 M 部分对于小数部分最多可以保留 52 位，多余的部分需要采用「四舍六入五取偶」的舍入策略，会导致精度的丢失。而在相加的过程中，因为`0.1`和`0.2`的阶码不一样需要对阶，使得尾数右移而导致精度再次丢失。

所以，转换和对阶都可能使得精度丢失。

为了更方便地进行转换和展示，做了一个转换函数：

```js
// 浮点数转二进制字符串
function DoubleToIEEE(decimal) {
    var buf = new ArrayBuffer(8);
    (new Float64Array(buf))[0] = decimal;
    const hi = new Uint32Array(buf)[1].toString(2).padStart(32, '0')
    const lo = new Uint32Array(buf)[0].toString(2).padStart(32, '0')
    const sign = hi.slice(0, 1)
    const exponent = hi.slice(1, 12)
    const mantissa = `${hi.slice(12)}${lo}`
    return `${sign} ${exponent} ${m}`
}
// 尾数累加，转换为小数
function processMantissa(mantissaStr){
    const LEN = 52
    const arr = mantissaStr.split('')
    return arr.reduce((total, num, index) => {
        return total+= +num * Math.pow(2, -(index+1))
    }, 0)
}
// 二进制字符串转浮点数
function IEEEToDouble(binaryStr){
   const arr = binaryStr.split(' ')
   let sign, exponent, mantissa
   if(arr && arr.length === 3){
     sign = arr[0]
     exponent = arr[1]
     mantissa = arr[2]
   } else {
     sign = binaryStr.slice(0, 1)
     exponent = binaryStr.slice(1, 12)
     mantissa = binaryStr.slice(12)
   }
   return (
       Math.pow(-1, parseInt(sign, 2))
       * Math.pow(2, parseInt(exponent, 2) - 1023)
       * (1 + processMantissa(mantissa))
   )
}
// 尾数累加，转换为小数
function processMantissa(mantissaStr){
    const LEN = 52
    const arr = mantissaStr.split('')
    return arr.reduce((total, num, index) => {
        return total+= +num * Math.pow(2, -(index+1))
    }, 0)
}
console.log(`
   ${DoubleToIEEE(0.1)}
+  ${DoubleToIEEE(0.2)}
=  ${DoubleToIEEE(0.1+0.2)}
`)
console.log(IEEEToDouble("0 01111111101 0011001100110011001100110011001100110011001100110100"))
```
#### 题外话

那为什么 C 语言中的双精度浮点数运算的结果就等于`0.3`呢？

``` c
int main()
{
  double a = 0.1;
  double b = 0.2;
  printf("%lf",a+b); // 0.3
}

```

这是因为`%lf`的完整格式为`%a.bf`，其中`a`为输出数据的宽度，默认无限制；`b`为输出数据的精度，默认值为`6`，所以打印了`0.3`，而当提高输出精度` printf("%.17f", a+b)`就会打印`0.30000000000000004`

### 如何用`Math.pow()`去模拟`Number.MIN_VALUE`等 Number 的常量？

经过前面「知识储备-Number 中的常量」章节的学习，就可以很容易得出结果了。在这里再总结一下：

- `Number.EPSILON`：表示 1 和`Number`可表示大于 1 的最小浮点数的差值，大小为`2^-52`
- `Number.MAX_SAFE_INTEGER`：最大的安全整数，大小为`2^53 - 1`
- `Number.MIN_SAFE_INTEGER`：最小的安全整数，大小为`-(2^53 - 1)`
- `Number.MAX_VALUE`：表示双精度浮点数能表示最大的值，**接近于**`2^1024`。但是`2^1024`为`Infinity`及`Infinity`减去一个数依然为`Infinity`，所以不能直接用`2^1024 - 2^971`表示，而是使用`2^1023 * (1+(2^0 - 2^-52))`。
- `Number.MIN_VALUE`：表示 JavaScript 能表示的最小正数，无限地接近 0，等于`2^-1074`
- `Number.POSITIVE_INFINITY`：正无穷，大于等于`2^1024`
- `Number.NEGATIVE_INFINITY`：负无穷，小于等于`-2^1024`

```js
Math.pow(2, -52) === Number.EPSILON
Math.pow(2, 53) - 1 === Number.MAX_SAFE_INTEGER
- (Math.pow(2, 53) - 1) === Number.MIN_SAFE_INTEGER
Math.pow(2, 1023) * (2 - Math.pow(2, -52))) === Number.MAX_VALUE
Math.pow(2, -1074) === Number.MIN_VALUE
Math.pow(2, 1024) === Number.POSITIVE_INFINITY
- Math.pow(2, 1024) === Number.NEGATIVE_INFINITY

```

### `toPrecision()`、`toFixed()`以及`Math.round()`的区别是什么？

#### toPrecision() 和 toFixed() 的区别

1）用途不同，`toPrecision`适用于处理精度显示，而`toFixed`适用于取整。

2）`toFixed(digit)`是取小数点后面第 n 位，`toPrecision(digit)`是取 n 位**有效数字**（从第一个非零的数字算起的所有数字）

3）对于`toFixed(digit)`digit 取值范围至少为 0，最大值在最新规范中为 100。也就是 `0 <=digit<=100`。digit 忽略的时候默认为 0；对于`toPrecision(digit)`，目前取值范围为 `1<=digit<=100`，如果 digit 忽略，则表现为直接输出字符串。两者的 digit 如果不为整数，均会进行向下取整，NaN 当作`+0`处理（ES 规范的`ToInteger`方法决定的）。

4）`toFixed()`的调用数字达到`1e+21`的数量级则会以简单地调用`Number.prototype.toString`进行输出科学计数法字符串。`toPrecision(digit)`的 digit 数字小于调用数字的整数位，则会输出科学计数法字符串（从科学计数法的角度来看，整数位数大于有效数字的位数必然导致科学计数法的出现）。或者`toPrecision(digit)`的结果数字数量级达到`1e-7`

#### toFixed() 和 Math.round()

`toFixed`和`Math.round()`都是用于取整。`Math.round()`是日常生活中使用的「四舍五入」且是转换成整数，而`Number.prototype.toFixed()`则是上面说到的「四舍六入五取偶」。以保留 `1.335` 的小数点后两位为例，可以看到舍入的策略有所不同。

```js
+(1.335).toFixed(2) // 1.33
// 因为 Math.round 是处理整数的舍入，所以得先进行乘法然后再使用除法
Math.round(1.335 * 100) / 100 // 1.34
```

### 为什么`(1.33500000000000001).toFixed(2) === '1.33'`而不是`'1.34'`

可以利用`(1.335).toPrecision(20)`来查看它更加精确的值，发现其实`1.335`经过转换为双精度浮点数存储后（精度丢失），实际上代表的是`1.3349999999999999645`。而且`Number.prototype.toFixed()`的舍入策略是「四舍六入五取偶」，要取小数点后 2 位则需要看小数点后第 3 位，也就是`4`，舍弃。所以结果为`1.33`而不是`1.34`。

### 相等判定与数量级问题

先来看一下相关的问题：

- 为什么`Number.MAX_SAFE_INTEGER + 1 === Number.MAX_SAFE_INTEGER + 2`
- 为什么`0.100000000000000002 === 0.100000000000000010`？
- 大于`Number.MAX_VALUE`的数表示无穷大，那为什么`Number.POSITIVE_INFINITY !== Number.MAX_VALUE + 1`？

// TODO:

## 总结

// TODO:

---

## 参考及工具

* [该死的 IEEE-754 浮点数，说「约」就「约」，你的底线呢？以 JS 的名义来好好查查你](https://juejin.im/entry/58f484af570c350056410bc8)
* [JavaScript 浮点数陷阱及解法](https://github.com/camsong/blog/issues/9)
* [Double-precision floating-point format - Wikipedia](https://en.wikipedia.org/wiki/Double-precision_floating-point_format)
* [javascript 的精度问题](https://fatshen3.cn/2018/05/29/javascript-float/)
* [Number.EPSILON 及其它属性 – cselftrain](https://cselftrain.wordpress.com/2016/11/15/number-epsilon%E5%8F%8A%E5%85%B6%E5%AE%83%E5%B1%9E%E6%80%A7/)

* [小数转换浮点数可视化工具](http://www.binaryconvert.com/convert_double.html)
* [IEEE 754 在线计算器](http://weitz.de/ieee/)
* [big.js: 用于任意精度的十进制运算库](https://github.com/MikeMcl/big.js/)