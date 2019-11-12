## flex 相关

flex 中有`container`以及`items`的概念，其中`container`是使用了`display: flex`的 DOM 节点，而`items`则是`container`的直接子元素。

flex 中还有主轴`the main axis`和横轴`the cross axis`的概念。主轴表示的是`flex-direction`指定的方向，而于主轴垂直的则为交叉轴。

### item 的相关属性

在 item 使用的 flex 相关属性较少，先来看一下 item 的 flex 相关属性：

```scss
.item {
  order: <integer>; // 控制在 container 中（根据 flex-direction 指定方向）出现的顺序，默认值为 0，取值为整数
  flex-grow: <number>; // 对于**剩余空间**的平等划分，默认值为 0，取值为数字（包括小数），负值无效
  flex-shrink: <number>; // 在需要的时候进行大小缩小，默认值为 1，取值为数字，负值无效
  flex-basis: <length>; // 定义「剩余空间分配」前的该元素占据主轴的大小，默认值为`auto`，取值为所有长度单位比如`20%`和`5rem`等。
  flex: none | [ <'flex-grow'> <'flex-shrink'>? || <'flex-basis'> ]; // 简写形式。
  align-self: auto | flex-start | flex-end | center | baseline | stretch; // 覆盖 align-items 的配置，单独配置某个 item 的对齐方式
}
```

当使用`display: flex`构造出 flex container 后，flex 元素默认有以下行为：

- 元素从主轴的起始线处开始，排列成一行，`flex-wrap`的值为`no-wrap`
- 元素在主维度上不会拉伸，但是可以缩小（全部元素的长度超出主轴长度时）
- 元素在交叉轴被拉伸填满
- `flex-basis`的值为`auto`

#### flex-basis

> By default flex items don't shrink below their minimum content size. To change this, set the item's min-width or min-height.

默认情况下，flex 不会缩小 item 到其最小内容大小以下。要想达到那种效果，请设置 item 的`min-width`或者`min-height`

#### flex

flex 是`flex-grow`、`flex-shrink`、`flex-basis`三个属性的简写，语法如下：

```scss
// 1. 关键字
flex: auto; // 相当于：`flex: 1 1 auto`，会伸长并吸收 flex 容器中额外的自由空间，也会缩短自身来适应 flex 容器
flex: initial; // 相当于：`flex: 0 1 auto`，缩短自身以适应 flex 容器，但不会伸长并吸收 flex 容器中的额外自由空间来适应 flex 容器，默认行为
flex: none; // 相当于：`flex: 0 0 auto`，既不会缩短，也不会伸长来适应 flex 容器
// 2. 单个无单位的数字，对应`flex-grow`
flex: 2;
// 3. 单个有长度单位的值，对应`flex-basis`
flex: 10em;
// 4. 两个值，第一个必须为`flex-grow`，没有单位的数字
// 4.1 第二个没有单位，对应`flex-grow`+`flex-shrink`
flex: 2 2;
// 4.2 第二个有单位，对应`flex-grow`+`flex-basis`
flex: 1 30px;
// 5. 三个值，对应`flex-grow`+`flex-shrink`+`flex-basis`，顺序不能错
flex: 2 2 10%;
```

`flex-grow`、`flex-shrink`以及`flex-basis`这三个属性的作用，都围绕着「avaliable space」，也可以称作「剩余空间」，控制着如何去拓展填充。

剩余空间怎么计算？`剩余空间 等于 container 的主轴长度 减去 所有主轴上 items 的总长度（flex-basis）`，_我们这里讲的剩余空间可能为负数，负数表示压缩，正数表示拉伸_

`flex-basis` 定义来其本身缩放之前的基准值。如果`flex-basis`为默认值`auto`，则查找看看自己本身在**主轴上**的大小是否确定，有则采用该大小，无则采用元素内容的尺寸。如果有确切的值则使用确切的值，且有确切值的`flex-basis`优先级比`width`或`height`的优先级高。

`flex-grow`和`flex-shrink` 是以`flex-basis`为基础，与其他 items 的`flex-grow`或`flex-shrink`进行剩余空间的分配。

`flex-grow`和`flex-shrink`的拉伸量或压缩量的计算不仅仅是分配比例的计算，还涉及了「分配的剩余空间之和」，「分配的剩余空间之和」乘以「分配比例」才是每个 item 真正的变化量。`flex-grow`和`flex-shrink`的分配比例计算有所不同，但是两者「分配的剩余空间之和」的概念是一样的 -- 如果所有的`flex-grow`或`flex-shrink`的系数之和大于等于 1，那么「分配的剩余空间之和」为全部的剩余空间；如果系数之和小于或等于 1，那么「分配的剩余空间之和」为「全部剩余空间 * 系数之和」。

而关于`flex-grow`和`flex-shrink`各个 item 的分配比例，计算如下：

```plain
// flex-grow
当前 item 分配比例 = 当前 item 的 flex-grow 系数 / 全部 items 的 flex-grow 系数之和

// flex-shrink -- 加权分配比例
总权重 = sum（各个 item 的宽度 * 各个 item 对应的 flex-shrink 系数）
当前 item 分配比例 = （当前 item 宽度 * 当前 item 的 flex-shrink 系数）/ 总权重
```

特别地，当 item 宽度都一样的时候，`flex-shrink`的分配比例计算与`flex-grow`一样。

### container 的相关属性

```scss
.container {
  display: flex;
  flex-direction: row | row-reverse | column | column-reverse; // 方向
  flex-wrap: nowrap | wrap | wrap-reverse; // 是否换行，wrap-reverse 的行为是上下调换，比如`wrap`时第一行 6 个，第二行 2 个，则`wrap-reverse`时第一行 2 个，第二行 6 个
  flex-flow: <‘flex-direction’> || <‘flex-wrap’>; // flex-direction 以及 flex-wrap 的简写形式
  justify-content: flex-start | flex-end | center | space-between | space-around | space-evenly; // 主轴的对齐方式（这里的空白间隙为 分配剩余空间后依然剩余的空间），默认值为 flex-start
  align-items: stretch | flex-start | flex-end | center | baseline; // 交叉轴上的对齐方式，默认值为 stretch
  align-content: flex-start | flex-end | center | space-between | space-around | space-evenly | stretch; // 定义了多根轴线在**交叉轴**的对齐方式。如果只有一根轴线，该属性不起作用
}
```

`justify-content`属性的常见值展示图片如下：

![justify-content 的常见值](http://image.geekaholic.cn/20191112170026.png@0.8)

其中，`space-around`和`space-evenly`的区别在于`space-aroung`是每一个 item 的左右间距都相同，但是不会合并间距，所以视觉上 item 的左右间距并不相等。而`space-evenly`则是视觉上的左右间距相等。

`align-items`属性的常见值展示图片如下：

![align-items 的常见值](http://image.geekaholic.cn/20191112170403.png@0.8)

`align-content`属性的常见值展示图片如下，与`justify-content`的值多了一个`stretch`：

![align-content 常见的属性值](http://image.geekaholic.cn/20191112170658.png@0.8)

## Grid 相关

Grid 布局是二维布局，使用网格线（grid line）将容器（container）划分为水平区域的行（row）和垂直区域的列（column），行和列统称为轨道（track）。行与列的交叉区域进而产生单元格（cell），这是 Grid 布局中最小可用空间单位。单元格与单元格之间的空间称为间隙（gap）。一个或多个单元格组成的矩形被称为区域（area），也就是「以四条网格线为边界的区域空间」称为「grid area」。

而 Flex 布局是一维布局，指定的是 items 相对轴线的位置。

### item 的相关属性

// TODO:

### container 的相关属性

```scss
.container {
  display: grid | inline-grid; // 块级或行级的 grid

  grid-template-columns: <track-size> ... | <line-name> <track-size> ...; // 用空格分隔的值列表定义网格的列
  grid-template-rows: <track-size> ... | <line-name> <track-size> ...; // 用空格分隔的值列表定义网格的行
  grid-template-areas: none | <string>+; // 用于命名 grid areas，与`grid-area`属性搭配使用
  grid-template: none | <grid-template-rows> / <grid-template-columns>; // 以上三个属性的缩写，但因为没有重置一些重要且常用的属性（`grid-auto-columns`、`grid-auto-rows`与`grid-auto-flow`），所以不推荐使用，不展开

  grid-column-gap: <line-size>; // 列与列之间的间隙，新名称为 column-gap，为了兼容，应该两个都写
  grid-row-gap: <line-size>; // 行与行之间的间隙，新名称为 row-gap，为了兼容，应该两个都写
  grid-gap: <grid-row-gap> <grid-column-gap>; // 简写形式，新名称为 gap，为了兼容，应该两个都写。如果省略第二个值，则第二个值与第一个值相同

  grid-auto-flow: row | column | row dense | column dense; // 控制 items 的放置方式

  justify-items: start | end | center | stretch; // 设置**单元格内容**的水平对齐。**stretch 为默认值**，表示单元格内容水平方向上全部占满
  align-items: start | end | center | stretch; // 设置**单元格内容**的垂直对齐，stretch 为默认值
  place-items: <align-items> <justify-items>; // align-items 与 justify-items 的缩写，注意顺序

  justify-content: start | end | center | stretch | space-around | space-between | space-evenly; // 设置 grid 在 grid container 的水平对齐（grid container 的宽度大于 grid 的时候有效，常见于 grid 使用了`px`等确切单位），stretch 为默认值
  align-content: start | end | center | stretch | space-around | space-between | space-evenly;// 设置 grid 在 grid container 的垂直对齐（grid container 的高度大于 grid 的时候有效，常见于 grid 使用了`px`等确切单位），stretch 为默认值
  place-content: <align-content> <justify-content>; // align-content 与 justify-content 的缩写，注意顺序
}
```

#### grid-template-columns 和 grid-template-rows

`grid-template-columns`和`grid-template-rows`用空格分隔的值列表定义网格的列和行。这些值表示`track`（两条`grid-line`之间的空间）大小，而且在声明的时候可以指定行和列中`grid-line`的名称，用`[]`裹住名称且可以包裹多个别名，方便之后的引用。

```scss
.container {
  grid-template-columns: [first] 40px [line2] 50px [line3] auto [col4-start] 50px [five] 40px [end];
  grid-template-rows: [row1-start] 25% [row1-end] 100px [third-line row2-start] auto [last-line];
}
```

上面代码的效果如下：

![](http://image.geekaholic.cn/20191112174212.png@0.8)

`track-size`除了常规的长度单位和百分比外，还有一些取值或方法值得我们关注：

- `repeat([<positive-integer> | auto-fill], <track-list>)`：用于简单化重复的值。第一个参数为重复次数，取值为正整数或`auto-fill`，`auto-fill`代表重复尽可能多次以填充 container。其中`<track-list>`就是`<track-size> ... | <line-name> <track-size> ...`，也就是`grid-template-columns`平常使用的值。比如`grid-template-columns: repeat(2, 100px 20px 80px);`定义了 6 列，第一和第四列为 `100px`，以此类推。
- `fr`单位的值：表示除去确切的宽度大小后的剩余空间，进行比例划分。比如` grid-template-columns: 150px 1fr 2fr;`表示第一列`150px`，第二列和第三列等比划分剩余空间，且第三列是第二列的 2 倍
- `auto`关键字：表示除去确切宽度大小后，尽可能为剩余空间的最大宽度。
- `minmax(min, max)`：定义大小范围为`[min, max]`，都是闭区间。如果原最大值小于原最小值，则原最大值为现最小值，没有现有最大值。且`max`可以使用`fr`单位进行弹性控制。

#### grid-template-areas

```scss
/* Keyword value */
grid-template-areas: none;

/* <string> values */
grid-template-areas: "a b"; // 两个区域，区域 a 和 区域 b
grid-template-areas: "a b b"
                     "a c ."; // 三列两行共六个单元格，分为 4 个区域，其中 a 和 b 各占两块，c 占一块，而第 4 个区域为英文点号，不需要利用不属于任何区域（empty）

```

举个例子：

```scss
.container {
  display: grid;
  grid-template-columns: 50px 50px 50px 50px;
  grid-template-rows: auto;
  grid-template-areas:
    "header header header header"
    "main main . sidebar"
    "footer footer footer footer";
}
/**
效果如图所示：

+----------------+--------------------+
|                |                    |
|               header                |
|                |                    |
+-------+------------------+----------+
|       |        |         |          |
|      main      |  empty  | sidebar  |
|       |        |         |          |
+-------+------------------+----------+
|                |                    |
|              footer                 |
|                |                    |
+----------------+--------------------+

**/
```

注意，区域的命名会影响到网格线。每个区域的起始网格线，会自动命名为`区域名-start`，终止网格线自动命名为`区域名-end`。这可能会导致一个网格线有多个名称。

比如，区域名为 `header`，则起始位置的水平网格线和垂直网格线叫做 `header-start`，终止位置的水平网格线和垂直网格线叫做 `header-end`。而如果`header`与`nav`都是最左边的区域，那最左侧的垂直网格线既可以是`header-start`也可以是`nav-start`

#### grid-auto-flow

有 4 个取值，分别为`row`、`column`、`row dense`以及`column dense`，分别表示`先行后列`、`先列后行`、`先行后列，且只要后面 item 符合前面的行空隙则填充`以及`先列后行，且后面 item 符合前面的列空隙则填充`

假设有 9 个 item，而 item1 和 item2 分别占据两个单元格`grid-column-start: 1;grid-column-end: 3;`，那么`grid-auto-flow`的四种情况展示如下图所示：

![4 种情况](http://image.geekaholic.cn/20191112205435.png@0.8)

可以看到`grid-auto-flow: row`和`grid-auto-flow: row dense`的区别。默认情况下则 item3 是在 item2 后面的，符合顺序情况。而如果设置为`grid-auto-flow: row dense`则后面的 item 尽可能地填充前面的空隙，这也是为什么 item3 会移到 item2 后面的原因。

### grid-auto-columns 与 grid-auto-rows

// TODO:

---

参考：

* [CSS Grid Terminology You Need to Know](https://www.lauraleeflores.com/blog/css-grid-terminology)
* [A Complete Guide to Flexbox | CSS-Tricks](https://css-tricks.com/snippets/css/a-guide-to-flexbox)
* [A Complete Guide to Grid | CSS-Tricks](https://css-tricks.com/snippets/css/complete-guide-grid/)
* [CSS Grid 网格布局教程 - 阮一峰的网络日志](https://www.ruanyifeng.com/blog/2019/03/grid-layout-tutorial.html)
* [详解 flex-grow 与 flex-shrink](https://github.com/xieranmaya/blog/issues/9)