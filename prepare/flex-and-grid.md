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

`flex-basis` 定义来其本身缩放之前的基准值，指定了 flex 元素在主轴方向上的初始大小。

如果`flex-basis`为默认值`auto`，则查找看看自己本身在**主轴上**的`width`或`height`大小是否确定（look at my width or height property），有则采用该大小，无则采用元素内容的尺寸。

如果`flex-basis`有确切的值则使用确切的值，且有确切值的`flex-basis`优先级比`width`或`height`的优先级高。如果`width`、`min-width`以及`max-width`都没有确切的值，那么这种情况`flex-basis`依然会随着内容的增多而变大，其余情况都不会随内容自动增长；

`flex-basis`会受到`min-width`的影响。初始值取两者中最大的，即`max(min-width, flex-basis)`，且不会随着内容的增多而增大。也会受到`max-width`的影响，最大值不超过`max-width`

> By default flex items don't shrink below their minimum content size. To change this, set the item's min-width or min-height.

默认情况下（`flex: auto`的情况），flex 不会缩小 item 到其最小内容大小以下。要想达到那种效果，请设置 item 的`min-width`或者`min-height`（则会缩小到`min-width`设置的宽度）。

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

以上的专有名词（网格线、网格、列、行、轨道、单元格、区域等）有一对修饰词，分别是隐式（implicit）和显式（explicit）。显式的网格是通过`grid-template-columns`、`grid-template-rows`、`grid-template-area`或融合了这几个语法的简写`grid-template`或`grid`进行创建。而隐式的网格是通过 grid item 的声明放置位置超出显式网格的时候或者 grid items 个数多于显式网格时创建，使用的语法是`grid-auto-columns`、`grid-auto-rows`或融合了这两者语法的`grid`简写形式。

而 Flex 布局是一维布局，指定的是 items 相对轴线的位置。

### item 的相关属性

`float`、`display: inline-block`、`display: table-cell`、`vertical-align`以及`column-*`属性在 grid item 中无效

```scss
// <grid-line> = auto | <custom-ident> | [ <integer> && <custom-ident>? ] | [ span && [ <integer> || <custom-ident> ] ]
// `auto` 关键字：表示自动放置，或者默认跨度（default span）为 1
// `<custom-ident>`：指的是网格线前缀，比如`foo`名称匹配`foo-start`与`foo-end`网格线（看属性是`*-start`还是`*-end`决定）。如果有多个`foo-start`和`foo-end`，默认为`1 foo`，也就是「第一个 foo」网格线，下面的语法会讲到。
// `<integer> && <custom-ident>?` ：表示「第 integer 个 custom-ident 网格线」。integer 为非 0 整数，如果为负数，从后面的显式网格线（explicit grid）数起。如果`<custom-ident>`有值，则表示「第 interger 个名称前缀为 custom-ident 的网格线」。如果没有足够的具有该名称的线，则假定所有的隐式网格线（implicit grid lines）都是该名称，然后再寻找（会自动创建隐式网格线）。
// `span && [ <integer> || <custom-ident> ]` : 表示跨度，一般用于`*-end`而不是`*-start`属性。`<interger>` 为正数，默认值为 1，和`span`组合表示所占据的单元格个数。而如果存在`<custom-ident>`，使用了语法`span && iteger && custom-ident`，则表示「从`start`位置开始数起（注意，不是从第一个开始数起），跨度到 interger 个（而不是第 interger 个）<custom-ident>-end 网格线」 -- 如果 interger 省略，则取默认值 1；如果没有足够的具有该名称的网格线，则假定**与搜索方向相同**的所有隐式网格线都是该名称，然后再寻找（会自动创建隐式网格线）。

// grid-area 的说明：控制 4 个边界的取值，分别对应`grid-row-start`、`grid-column-start`、`grid-row-end`和`grid-column-end`（边界上->左->下->右）。
// TL;DR：总的来说看对应的值来决定缺失的值

// - 如果`grid-column-end`为空（3 个值的时候）：如果`grid-column-start`是`custom-ident`，那么`grid-column-end`为``custom-ident``，否则为`auto`
// - 如果`grid-row-end`和`grid-column-end`为空（2 个值的时候）：如果`grid-row-start`为`custom-ident`，`grid-row-end`为`custom-ident`，否则为`auto`。至于`grid-column-end`的取值逻辑一样。
// -如果`grid-column-start`、`grid-row-end`和`grid-column-end`为空（一个值的时候）：如果为`custom-ident`，4 个值都为`custom-ident`，这时候表示取的`custom-ident`整个区域；否则为`auto`

.item {
  grid-column-start: <grid-line>;
  grid-column-end: <grid-line>;
  grid-row-start: <grid-line>;
  grid-row-end: <grid-line>;
  grid-column: <grid-line> / <grid-line>;
  grid-row: <grid-line> / <grid-line>; // 以上的属性都是用于控制 item 的放置区域，包括位置和跨度
  grid-area: <grid-line> [ / <grid-line> ]{0, 3};

  justify-self: start | end | center | stretch; // 默认值为 stretch，设置单个单元格内部的水平位置，用于覆盖 container 的 justify-items
  align-self: start | end | center | stretch; // 默认值为 stretch，设置单个单元格内部的垂直位置，用于覆盖 container 的 align-items
  place-self: <align-self> <justify-self>; // align-self 与 justify-self 的缩写，注意顺序，如果第二值省略则两个取值都为第一个值

}
```

### container 的相关属性

```scss
.container {
  display: grid | inline-grid; // 块级或行级的 grid

  grid-template-columns: <track-size> ... | <line-name> <track-size> ...; // 用空格分隔的值列表定义网格的列
  grid-template-rows: <track-size> ... | <line-name> <track-size> ...; // 用空格分隔的值列表定义网格的行
  grid-template-areas: none | <string>+; // 用于命名 grid areas，与`grid-area`属性搭配使用
  grid-template: none | [ <'grid-template-rows'> / <'grid-template-columns'> ] | [ <line-names>? <string> <track-size>? <line-names>? ] + [ /<explicit-track-list> ]?; // 以上三个属性的缩写，但因为没有重置隐式网格的属性（`grid-auto-columns`、`grid-auto-rows`与`grid-auto-flow`），所以不推荐使用，语法的讲解在简写`grid`章节中展开

  grid-column-gap: <line-size>; // 列与列之间的间隙，新名称为 column-gap，为了兼容，应该两个都写
  grid-row-gap: <line-size>; // 行与行之间的间隙，新名称为 row-gap，为了兼容，应该两个都写
  grid-gap: <grid-row-gap> <grid-column-gap>; // 简写形式，新名称为 gap，为了兼容，应该两个都写。如果省略第二个值，则第二个值与第一个值相同

  grid-auto-flow: row | column | row dense | column dense; // 控制 items 的放置方式

  grid-auto-columns: <track-size> ...; // 当 grid items 多于 grid 的单元格或将 grid item 放置在定义的 grid 之外时，会创建隐式轨道（implicit track）。`grid-auto-columns`与`grid-auto-rows`用来指定 implicit track 的大小。
  grid-auto-rows: <track-size> ...; // 同上

  justify-items: start | end | center | stretch; // 设置**单元格内容**的水平对齐。**stretch 为默认值**，表示单元格内容水平方向上全部占满
  align-items: start | end | center | stretch; // 设置**单元格内容**的垂直对齐，stretch 为默认值
  place-items: <align-items> <justify-items>; // align-items 与 justify-items 的缩写，注意顺序，如果第二值省略则两个取值都为第一个值

  justify-content: start | end | center | stretch | space-around | space-between | space-evenly; // 设置 grid 在 grid container 的水平对齐（grid container 的宽度大于 grid 的时候有效，常见于 grid 使用了`px`等确切单位），stretch 为默认值。`space-*`的情况针对 grid columns 进行间隙隔开。
  align-content: start | end | center | stretch | space-around | space-between | space-evenly;// 设置 grid 在 grid container 的垂直对齐（grid container 的高度大于 grid 的时候有效，常见于 grid 使用了`px`等确切单位），stretch 为默认值。`space-*`的情况针对 grid rows 进行间隙隔开。
  place-content: <align-content> <justify-content>; // align-content 与 justify-content 的缩写，注意顺序，如果第二值省略则两个取值都为第一个值

  grid: <'grid-template'> | <'grid-template-rows'> / [ auto-flow && dense? ] <'grid-auto-columns'>? | [ auto-flow && dense? ] <'grid-auto-rows'>? / <'grid-template-columns'>; // 在下边相关章节进行语法讲解
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

##### CSS 宽高度新增的取值

内容来自 [CSS3 四个自适应关键字——fill-available、max-content、min-content、fit-content](https://www.cnblogs.com/xiaohuochai/p/7210540.html)

在讲`max-content`、`min-content`之前，看一下 CSS3 中`width`属性新增的一些关键值，估计就更加容易理解了

- `fill-available`: 表示撑满可用空间，可以让元素的 100%自动填充特性不仅仅在 block 元素上，也可以应用在其他元素比如`inline-block`（利用此属性可以轻松实现等高布局）
- `fit-content`：将元素宽度收缩为内容宽度，结合`margin: auto`可以实现水平剧中效果
- `min-content`：采用内部元素最大的「最小宽度值」作为最终的容器宽度。首先，要明白这里的「最小宽度值」是什么意思。替换元素，例如图片的最小宽度值就是图片呈现的宽度，对于文本元素，如果全部是中文，则最小宽度值就是一个中文的宽度值；如果包含英文，因为默认英文单词不换行，所以，最小宽度可能就是里面最长的英文单词的宽度
- `max-content`：采用内部元素宽度值最大的那个元素的宽度作为最终容器的宽度。如果出现文本，则相当于文本不换行。

##### track-size 的取值

`track-size`除了常规的长度单位和百分比外，还有一些取值或方法值得我们关注：

- `none`关键字：表示没有显式 grid，所有的行（或列）都是隐式 grid，并且大小由`grid-auto-rows`（或`grid-auto-columns`）决定。
- `repeat([<positive-integer> | auto-fill | auto-fit], <track-list>)`：用于简单化重复的值。第一个参数为重复次数，取值为正整数或`auto-fill`，`auto-fill`代表重复尽可能多次以填充 container，而`auto-fit`指的是先`auto-fill`，然后合并空的列（或行）。`auto-fill`与`auto-fit`更详细的区别见下面相关章节。其中`<track-list>`就是`<track-size> ... | <line-name> <track-size> ...`，也就是`grid-template-columns`平常使用的值。比如`grid-template-columns: repeat(2, 100px 20px 80px);`定义了 6 列（3 列重复了 2 次），第一和第四列为 `100px`，以此类推。
- `<flex>`：以`fr`为单位，表示除去确切的宽度大小后的剩余空间，进行比例划分。比如` grid-template-columns: 150px 1fr 2fr;`表示第一列`150px`，第二列和第三列等比划分剩余空间，且第三列是第二列的 2 倍
- `auto`关键字：取决于内容宽度（文本正常断行），最小最大值看自身内容宽度以及当前所在的 grid track 的最小最大值。比如`grid-template-columns: auto 1fr 1fr 10%;`。第一列的宽度取决于其内容，但最小值取第一列各行内容「最小宽度值」的最大那一个（因为已经撑开了）；而最大值取第一列各行内容「最大宽度值」，如果自身值比最大值还大那就是自身宽度为最大值。
- `minmax([ <length> | <percentage> | <flex> | min-content | max-content | auto ] , [ <length> | <percentage> | min-content | max-content | auto ] )`：定义大小范围为`[min, max]`，都是闭区间。如果原最大值小于原最小值，则原最大值为现最小值，没有现有最大值，整体函数表现为`min`，也就是只限定最小值。且`max`可以使用`fr`单位进行弹性控制。比如`minmax(200px, auto)`表示，最小为`200px`，但是会根据内容的宽度进行改变。
- `fit-content([ <length> | <percentage> ])`：与`auto`类似，取决于内容宽度，最小最大值看自身内容宽度以及当前所在的 grid track 的最小最大值。不同之处在于，最大值取`min(max-content, max(auto, argument))`，其中`max-content`值的是句子不断行情况下最长的宽度，`auto`表示正常断行的情况，`argument`表示传入的参数。可以得知，如果整个内容是一个单词，那么宽度就是为`max-content`（因为 min 两个参数相等）；如果正常断行，那么看正常断行和传入的参数哪一个更大，取值更大的。（个人通过编写 demo 后的看法，暂未有规范支撑）

##### auto-fill vs auto-fit

内容来自 [[译] CSS Grid 之列宽自适应：`auto-fill` vs `auto-fit`](https://juejin.im/post/5a8c25d66fb9a0634d27b73b)：

只有行的宽度大到能够容纳额外的列时，`auto-fill` 和 `auto-fit` 这两者的区别才会体现出来。

`auto-fill`倾向于容纳更多的列，所以如果在满足宽度限制的前提下还有空间能容纳新列，那么它会暗中创建一些列来填充当前行。即使创建出来的列没有任何内容，但实际上还是占据了行的空间。

`auto-fit`倾向于使用最少列数占满当前行空间，浏览器先是和 auto-fill 一样，暗中创建一些列来填充多出来的行空间，然后坍缩（collapse）这些列以便腾出空间让其余列扩张（但是列依然创建了，通过默认的 grid line name 依然可以引用对应的网格线，只不过坍塌合并在了一起）。

`auto-fit` 的做法和 `auto-fill` 一样，随着视口宽度增大而增加列数，区别在于新增加的列都坍缩了（包括间隔 gap 在内）。用 Firefox 的 Grid 工具来可视化这个过程再合适不过了，当视口的宽度增加时，新的列也被添加进来，grid 的线条也会增加，肉眼就能观察得到全过程。（英文原文的视频可见）

![对比图](http://image.geekaholic.cn/20191114131302.jpg@0.8)

有必要记住的一点是，在以上两种情况中，多出来的列（无论最后是否坍缩）都不是隐式列（implicit columns）而是显式列。

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

比如，区域名为 `header`，则起始位置的水平网格线和垂直网格线叫做 `header-start`，终止位置的水平网格线和垂直网格线叫做 `header-end`。而如果`header`与`nav`都是最左边的区域，那最左侧的垂直网格线既可以是`header-start`也可以是`nav-start`。

除了`grid-template-areas`会自动命名网格线之外，还可以使用`grid-template-columns`以及`grid-template-rows`显式命名，或者默认就有名字（从`1`开始），如下图例子所示：

![命名网格线](http://image.geekaholic.cn/20191113205902.png@0.8)

#### grid-auto-flow

有 4 个取值，分别为`row`、`column`、`row dense`以及`column dense`，分别表示`先行后列`、`先列后行`、`先行后列，且只要后面 item 符合前面的行空隙则填充`以及`先列后行，且后面 item 符合前面的列空隙则填充`

假设有 9 个 item，而 item1 和 item2 分别占据两个单元格`grid-column-start: 1;grid-column-end: 3;`，那么`grid-auto-flow`的四种情况展示如下图所示：

![4 种情况](http://image.geekaholic.cn/20191112205435.png@0.8)

可以看到`grid-auto-flow: row`和`grid-auto-flow: row dense`的区别。默认情况下则 item3 是在 item2 后面的，符合顺序情况。而如果设置为`grid-auto-flow: row dense`则后面的 item 尽可能地填充前面的空隙，这也是为什么 item3 会移到 item2 后面的原因。

### grid-auto-columns 与 grid-auto-rows

当 grid items 多于 grid 的单元格或将 grid item 放置在定义的 grid 之外时，会创建隐式轨道（implicit track）。`grid-auto-columns`与`grid-auto-rows`用来指定 implicit track 的大小。

我们直接来看一个例子：

```scss
.container {
  grid-template-columns: 60px 60px;
  grid-template-rows: 90px 90px;
  grid-auto-columns: 60px; // <= 加上这一行
}
.item-a {
  grid-column: 1 / 2; // 从第一个 grid-line 到第二个 grid-line 之间
  grid-row: 2 / 3;
}
.item-b {
  grid-column: 5 / 6;
  grid-row: 2 / 3;
}
```

以上代码的表示：声明一个`2 * 2`的网格，将`item-a`放置在第 1 个列网格线-第 2 个列网格线以及第 2 个行网格线与第 3 个行网格线的交叉单元格（也就是第 2 行第 1 列的一个单元格），将`item-b`放置在第 2 行第 5 列的一个单元格处。

`item-b`已经超出了声明的 grid 范围，会自动生成新的隐式网格，用于放置`item-b`。在 Chrome 的测试中，自动生成的单元格的大小应该都默认为`auto`（表现为：与其他`auto`平分剩余空间）

![使用 grid-auto-columns 前](http://image.geekaholic.cn/20191113170716.png@0.8)

上图为使用`grid-auto-columns: 60px`之前，新增的 6 个单元格应该 columns 大小都为`auto`，而不是图中的`0 0 auto`。

![使用 grid-auto-columns 后](http://image.geekaholic.cn/20191113170551.png@0.8)

上图为使用`grid-auto-columns: 60px`后，新增的 6 个单元格 columns 大小都为`60px`。

### grid 与 grid-template

前面说到，`grid-template`是`grid-template-columns`、`grid-template-rows`以及`grid-template-areas`的简写形式，但是因为没有重置网格的属性（`grid-auto-columns`、`grid-auto-rows`与`grid-auto-flow`），所以不推荐单独使用。

而`grid`的简写十分之复杂，它是显式网格（explicit grid）属性`grid-template-rows`、`grid-template-columns`、`grid-template-areas`，以及隐式网格（implicit grid）属性`grid-auto-rows`、`grid-auto-columns`、`grid-auto-flow`的简写属性。

我们先来看一下语法：

```scss
// 先来看一下 grid-template，下边的 grid 有用到
grid-template: none | [ <'grid-template-rows'> / <'grid-template-columns'> ] | [ <line-names>? <string> <track-size>? <line-names>? ]+ [ / <explicit-track-list> ]?

// 取值情况
/* 1⃣️ Keyword  */
grid-template: none;

/* 2⃣️ grid-template-rows / grid-template-columns */
grid-template: 100px 1fr / 50px 1fr;
grid-template: auto 1fr / auto 1fr auto;
grid-template: [linename] 100px / [columnname1] 30% [columnname2] 70%;
grid-template: fit-content(100px) / fit-content(40%);

/* grid-template-areas grid-template-rows / grid-template-column values */
grid-template: "a a a"
               "b b b";
grid-template: "a a a" 20%
               "b b b" auto;
grid-template: [header-top] "a a a"     [header-bottom]
                 [main-top] "b b b" 1fr [main-bottom]
                            / auto 1fr auto; // <= grid-template-rows 要一一对应 grid-template-areas 中的区域，缺失的值为 auto（表示大小取决于内容）

```

```scss
grid: <'grid-template'> | <'grid-template-rows'> / [ auto-flow && dense? ] <'grid-auto-columns'>? | [ auto-flow && dense? ] <'grid-auto-rows'>? / <'grid-template-columns'>;

// 取值情况
/* 1⃣️ Global values */
grid: inherit;
grid: initial;
grid: unset;

/* 2⃣️ <'grid-template'> */
grid: none;
grid: <'grid-template'>; // grid-template 的语法见上边

/* 3⃣️ <'grid-template-rows'> /
   [ auto-flow && dense? ] <'grid-auto-columns'>? */
grid: 200px / auto-flow;
grid: 30% / auto-flow dense;
grid: repeat(3, [line1 line2 line3] 200px) / auto-flow 300px;
grid: [line1] minmax(20em, max-content) / auto-flow dense 40%;

/* 4⃣️ [ auto-flow && dense? ] <'grid-auto-rows'>? /
   <'grid-template-columns'> */
grid: auto-flow / 200px;
grid: auto-flow dense / 30%;
grid: auto-flow 300px / repeat(3, [line1 line2 line3] 200px);
grid: auto-flow dense 40% / [line1] minmax(20em, max-content);

```

从上边的语法可以看到，语法 3 和语法 4 是相似的，都是声明的隐式网格相关属性，不同在于`auto-flow`或`auto-flow dense`放在符号`/`的左侧还是右侧。

总体的原则是，放在哪一侧`grid-auto-flow`的取值就是哪一个。如果放在右侧，可以看到右侧为`<'grid-auto-columns'>`语法，则表示`grid-auto-flow: column`或`grid-auto-flow: column dense`；如果放在左侧，可以看到左侧为`<'grid-auto-rows'>`语法，则表示`grid-auto-flow: row`或`grid-auto-flow: row dense`；

值得注意的是，`auto-flow`或`auto-flow dense`放置的一侧语法为`<'grid-auto-*'>`，是隐式网格的属性；而另外一侧语法为`<'grid-template-*'>`，是显式网格的属性。

_可以看到，使用 grid 简写属性会十分之麻烦且不利于理解，所以尽量少使用_

---

参考：

* [CSS Grid Terminology You Need to Know](https://www.lauraleeflores.com/blog/css-grid-terminology)
* [A Complete Guide to Flexbox | CSS-Tricks](https://css-tricks.com/snippets/css/a-guide-to-flexbox)
* [A Complete Guide to Grid | CSS-Tricks](https://css-tricks.com/snippets/css/complete-guide-grid/)
* [CSS Grid 网格布局教程 - 阮一峰的网络日志](https://www.ruanyifeng.com/blog/2019/03/grid-layout-tutorial.html)
* [详解 flex-grow 与 flex-shrink](https://github.com/xieranmaya/blog/issues/9)