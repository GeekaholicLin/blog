# 常见排序算法

```js
// 可用于测试
// 产生随机数
var genRandomNums = () => {
  return [...Array(10)].map(_ => Math.floor(Math.random() * 100));
};
// bubbleSort 可替换成其他排序算法
var randomNums = genRandomNums();
var sortedNums = randomNums.sort((a, b) => a - b);
sortedNums.every((num, index) => num === bubbleSort(randomNums)[index]);
```

## 概述

- 比较类排序：通过比较来决定元素间的相对次序，由于其时间复杂度不能突破 O(nlogn)，因此也称为非线性时间比较类排序。
- 非比较类排序：不通过比较来决定元素间的相对次序，它可以突破基于比较排序的时间下界，以线性时间运行，因此也称为线性时间非比较类排序。

![排序算法分类](http://image.geekaholic.cn/20191126145019.png@0.8)
![算法复杂度概括](http://image.geekaholic.cn/20191126145354.png@0.8)

## 交换排序

### 冒泡排序

双重循环。循环数组，比较当前元素和下一个元素，如果当前元素比下一个元素大，交换使得元素大的往上冒泡。

![冒泡排序演示](https://images2017.cnblogs.com/blog/849589/201710/849589-20171015223238449-2146169197.gif)

```js
function bubbleSort(arr) {
  const len = arr.length;
  /* 外循环为**排序趟数**，len 个数进行 len-1 趟 */
  for (let i = 0; i < len - 1; ++i) {
    /* 内循环为比较数的下标 */
    for (let j = 0; j < len - 1 - i; ++j) {
      if (arr[j] > arr[j + 1]) {
        // 当前数大于下一个数，交换
        [arr[j], arr[j + 1]] = [arr[j + 1], arr[j]];
      }
    }
  }
  return arr;
}
```

### 鸡尾酒排序

是冒泡排序的一种变形，不同之处在于一次遍历，双向（从左到右再从右到左）比对且排序。

```js
const cocktailSort = arr => {
  let i;
  let left = 0;
  let right = arr.length - 1;
  while (left < right) {
    // 从左到右
    for (i = left; i < right; ++i) {
      if (arr[i] > arr[i + 1]) {
        [arr[i], arr[i + 1]] = [arr[i + 1], arr[i]];
      }
    }
    right--;
    for (i = right; i > left; --i) {
      if (arr[i - 1] > arr[i]) {
        [arr[i], arr[i - 1]] = [arr[i - 1], arr[i]];
      }
    }
    left++;
  }
  return arr;
};
```

### 快速排序

算法步骤：

1. 挑选基准值：从数列中挑出一个元素，称为 “基准”（pivot） ；
2. 分割：重新排序序列，所有比基准值小的元素摆放在基准前面，所有比基准值大的元素摆在基准后面（与基准值相等的数可以到任何一边）。在这个分割结束之后，对基准值的排序就已经完成；
3. 递归排序子序列：递归地将小于基准值元素的子序列和大于基准值元素的子序列排序。

![快速排序](https://images2017.cnblogs.com/blog/849589/201710/849589-20171015230936371-1413523412.gif)

```js
// 常规快排
const quickSort = arr => {
  const len = arr.length;
  if (len < 2) {
    return arr;
  }
  const pivot = arr[0];
  const left = [];
  const right = [];
  for (let i = 1; i < len; ++i) {
    if (arr[i] >= pivot) {
      right.push(arr[i]);
    }
    if (arr[i] < pivot) {
      left.push(arr[i]);
    }
  }
  return [...quickSort(left), pivot, ...quickSort(right)];
};
```

当面对一个有大量重复的数据的序列时，选取 pivot 的快速排序有可能会退化成一个 O(n^2) 的算法。所以就有了一个快速排序的优化版本，叫 「三路快排」-- 将序列分为三部分：小于 pivot，等于 pivot 和大于 pivot，之后递归的对小于 v 和大于 v 部分进行排序。

```js
const quickSort = arr => {
  const len = arr.length;
  if (len < 2) {
    return arr;
  }
  let left = [];
  let center = []; // 增加了一个 center 数组，用于存储与 pivot 相等的值
  let right = [];
  let pivot = arr[0];
  for (let i = 0; i < len; ++i) {
    if (arr[i] < pivot) {
      left.push(arr[i]);
    } else if (arr[i] === pivot) {
      center.push(arr[i]);
    } else {
      right.push(arr[i]);
    }
  }
  return [...quickSort(left), ...center, ...quickSort(right)];
};
```

上面的两个版本中需要额外的 O(n) 的空间复杂度，可以采用 in-place 版本实现原地快速排序：

```js
function quickSort(array) {
  // 数组分区，左小右大
  // 如果基准数在右边的话，扫描一定要从左边开始。
  // 这是为了保证基准数调换位置之后左边的都比基准小，右边的都比基准大
  function partition(left, right) {
    var storeIndex = left; // 存储小于 pivot 元素的下标，初始化为 left，也叫游标
    var pivot = array[right]; // 直接选最右边的元素为基准元素
    for (var i = left; i < right; i++) {
      if (array[i] < pivot) {
        [array[storeIndex], array[i]] = [array[i], array[storeIndex]];
        storeIndex++; // 交换位置后，storeIndex 自增 1，代表下一个可能要交换的位置
      }
    }
    [array[storeIndex], array[right]] = [array[right], array[storeIndex]]; // 将基准元素放置到最后的正确位置上
    return storeIndex;
  }
  function sort(left, right) {
    if (left > right) {
      return;
    }
    var storeIndex = partition(left, right); // 返回的 storeIndex 为基准值所在的下标
    sort(left, storeIndex - 1);
    sort(storeIndex + 1, right);
  }
  sort(0, array.length - 1);
  return array;
}
```

## 插入排序

### 简单插入排序

将左侧序列看成一个有序序列，从有序序列最右侧开始比较，**边比较边移动**，将比较的数插入到合适的位置。

![简单插入排序](https://images2017.cnblogs.com/blog/849589/201710/849589-20171015225645277-1151100000.gif)

```js
function insertionSort(arr) {
  const len = arr.length;
  let j, temp;
  for (let i = 1; i < len; ++i) {
    // 把第一个数当成有序，从第二个数开始，i 为要插入的数的下标
    j = i - 1;
    temp = arr[i]; // arr[i] 为要插入的数
    while (j >= 0 && arr[j] > temp) {
      // j 要有一个临界值判断
      // arr[j] 表示有序序列中的数，边比较边移动，一直找到不大于的位置
      arr[j + 1] = arr[j];
      j -= 1;
    }
    arr[j + 1] = temp; // arr[j] 等于小于 temp，所以在 a[j+1] 处插入
  }
  return arr;
}
```

### 希尔排序

希尔排序是因为「插入排序」的两个特点而进行改进的排序方法：

- 插入排序对几乎排好序的数据插入操作时，效率高，可以达到 O(n) 的时间复杂度
- 但插入排序的步长为 1，每次只能移动一位，效率较低

所以希尔排序引入了「步长」的概念，会优先比较距离较远的元素，步长的选择是希尔排序的重要部分。

希尔排序以一定步长开始，逐渐递减使得**最终步长**为 1，而步长为 1 时的排序即为插入排序。步长的定义可以通过动态去生成。

![希尔排序](https://images2018.cnblogs.com/blog/849589/201803/849589-20180331170017421-364506073.gif)

```js
// 从代码上看，相比于「插入排序」增加了
// 1. 外层的 gap 循环
// 2. 将代码中的 `1` 替换成步长 `gap`
function shellSort(arr) {
  var len = arr.length;
  // gap 选择为 n / 2 并且对 gap 取半直到 gap 达到 1
  for (var gap = Math.floor(len / 2); gap > 0; gap = Math.floor(gap / 2)) {
    let j, temp;
    for (let i = gap; i < len; ++i) {
      j = i - gap;
      temp = arr[i]; // arr[i] 为要插入的数
      while (j >= 0 && arr[j] > temp) {
        // arr[j] 表示有序序列中的数，边比较边移动，一直找到不大于的位置
        arr[j + gap] = arr[j];
        j -= gap;
      }
      arr[j + gap] = temp; // arr[j] 等于小于 temp，所以在 a[j+1] 处插入
    }
  }
  return arr;
}
```

## 选择排序

### 简单选择排序

双重循环。对于外围而言，每一个数的下标在遍历开始默认为最小值，将其与内围循环**未排序**的每一个数进行比较，记录较小数的下标，循环结束后进行交换。

![选择排序](https://images2017.cnblogs.com/blog/849589/201710/849589-20171015224719590-1433219824.gif)

```js
function selectionSort(arr) {
  // 注意 i 和 j 的循环终止条件不一样
  for (let i = 0; i < arr.length - 1; i++) {
    let minIndex = i; // 初始化最小下标为当前比较值
    for (let j = i + 1; j < arr.length; j++) {
      if (arr[j] < arr[minIndex]) {
        minIndex = j; // 更新最小下标
      }
    }
    // 遍历结束后，交换
    [arr[minIndex], arr[i]] = [arr[i], arr[minIndex]];
  }
  return arr;
}
```

### 堆排序

堆排序（Heapsort） 是指利用 二叉堆 这种数据结构所设计的一种排序算法。堆是一个近似 完全二叉树 的结构，并同时满足堆积的性质 ：即子节点的键值或索引总是小于（或者大于）它的父节点。

最大堆： 最大堆任何一个父节点的值，都大于等于它左右孩子节点的值。

![最大堆](http://image.geekaholic.cn/20191126165334.png@0.8)

最小堆：最小堆任何一个父节点的值，都小于等于它左右孩子节点的值。

![最小堆](http://image.geekaholic.cn/20191126165405.png@0.8)

堆和数组的对应关系：

1. 设某结点序号为 i, 则其父结点为`⌊i/2⌋`，2i+1 为左子结点序号，2i+2 为右子结点序号。其中，`⌊⌋`为向下取整符号
2. 当存储了 n 个元素时，`⌊n/2⌋+1`、`⌊n/2⌋+2`、···、n 为叶结点。

![对应关系图](http://image.geekaholic.cn/20191127130055.jpeg@0.8)

堆排序演示动画：

![堆排序](https://images2017.cnblogs.com/blog/849589/201710/849589-20171015231308699-356134237.gif)

```js
function heapSort(arr) {
  let size = arr.length;

  // 初始化堆，i 从最后一个父节点开始调整，直到节点均调整完毕
  for (let i = Math.floor(size / 2) - 1; i >= 0; i--) {
    heapify(arr, i, size);
  }
  // 堆排序：先将第一个元素和已拍好元素前一位作交换，再重新调整，直到排序完毕
  for (let i = size - 1; i > 0; i--) {
    // swap(arr, 0, i);
    [arr[0], arr[i]] = [arr[i], arr[0]];
    size -= 1; // size 减少
    heapify(arr, 0, size); // 对未排序的继续调整最大堆
  }

  return arr;
}

// 根据传入的父节点，进行堆调整
function heapify(arr, index, size) {
  let largest = index; // 默认父节点的下标为最大值的下标
  let left = 2 * index + 1;
  let right = 2 * index + 2;

  // 左叶子节点存在且数值大于父节点数字，交换下标
  if (left < size && arr[left] > arr[largest]) {
    largest = left;
  }
  // 同上
  if (right < size && arr[right] > arr[largest]) {
    largest = right;
  }
  // 如果不等，则交换数值且对交换后的子树进行堆调整
  if (largest !== index) {
    [arr[index], arr[largest]] = [arr[largest], arr[index]];
    heapify(arr, largest, size); // 递归调整，下标为最大的数对应的下标
  }
}
```

## 归并排序

### 二路归并排序

采用了「分而治之」的思想：

1. 将输入长度的序列一分为二，分为左子序列和右子序列
2. 分别对这两个子序列排序
3. 将排序好的子序列合并成一个序列后进行排序

![二路归并排序](https://images2017.cnblogs.com/blog/849589/201710/849589-20171015230557043-37375010.gif)

```js
// 二路归并的递归实现
// 划分并排序
function mergeSort(arr) {
  const len = arr.length;
  if (len < 2) {
    return arr;
  }
  const mid = Math.floor(len / 2); // 二分
  const left = arr.slice(0, mid);
  const right = arr.slice(mid);
  return merge(mergeSort(left), mergeSort(right));
}
// 排序并合并给定的两个子序列
function merge(left, right) {
  const result = [];
  // 一直到其中一个为空
  while (left.length > 0 && right.length > 0) {
    result.push(left[0] <= right[0] ? left.shift() : right.shift());
  }
  // 直接拼接剩下的部分 -- left 和 right 其中一个必然为空
  return result.concat(left, right);
}
```

```js
// 二路归并非递归实现 -- 非递归的部分指划分为两个子序列的方法，即 mergeSort
// 关键是步长成倍增加，如果步长大于或等于总数组长度说明合并结束！
// 合并和排序函数 merge 与非递归的一样，不需要改动

// 1. 把 n 个记录看成 n 个长度为 1 的有序子序列；
// 2. 进行两两归并使记录关键字有序，得到 n/2 个长度为 2 的有序子序列；
// 3. 重复第 2 步直到所有记录归并成一个长度为 n 的有序子序列为止。
function mergeSort(arr) {
  let result = [...arr];
  let len = result.length;
  let first, mid, last, left, right;
  for (let step = 1; step < len; step *= 2) {
    // step 为合并区间里其中一个子序列的长度
    for (let i = 0; i < len; i += 2 * step) {
      // i 为下标
      // i=i+2*step 表示跳到下一个需要合并的区间的第一个下标
      first = i; // 合并区间的第一个数的下标
      last = Math.min(i + 2 * step, len); // 合并区间的最后一个数的下标
      mid = first + step; // 不能使用 Math.floor((first+last)/2)，因为 last 不一定是 i+2*step，导致分割左右子序列出错
      left = result.slice(first, mid);
      right = result.slice(mid, last);
      result.splice(first, step * 2, ...merge(left, right));
    }
  }
  return result;
}
```

## 非比较类排序

非比较类排序中的「计数排序」、「桶排序」以及「基数排序」原理相似，都借助了额外的空间（或称为「桶」）。但三者有点差异：

- 基数排序：根据键值的每位数字来分配桶；
- 桶排序：每个桶存储一定范围的数值；
- 计数排序：每个桶只存储单一键值。

### 计数排序

计数排序使用一个额外的数组来存储输入的元素，计数排序要求输入的数据**必须是有确定范围的整数**。

步骤：

- 找出待排序的数组中最大和最小的元素；
- 统计数组中每个值为 i 的元素出现的次数，存入数组 C 的第 i 项；
- 对所有的计数累加（从 C 中的第一个元素开始，每一项和前一项相加）；
- 反向填充目标数组：将每个元素 i 放在新数组的第 C(i) 项，每放一个元素就将 C(i) 减去 1

![计数排序](https://images2017.cnblogs.com/blog/849589/201710/849589-20171015231740840-6968181.gif)

```js
const countSort = arr => {
  const C = [];
  for (let i = 0, iLen = arr.length; i < iLen; ++i) {
    const j = arr[i]; // 数值
    if (C[j] >= 1) {
      // 数值对应的下标的数组 C 进行计数
      C[j]++;
    } else {
      C[j] = 1;
    }
  }
  const D = [];
  // 遍历填充
  for (let j = 0, jLen = C.length; j < jLen; ++j) {
    if (C[j]) {
      while (C[j] > 0) {
        D.push(j);
        C[j]--;
      }
    }
  }
  return D;
};
```

### 桶排序

桶排序是「计数排序」的升级版。它利用了函数的映射关系，高效与否的关键就在于这个映射函数的确定。桶排序 (Bucket sort) 的工作的原理：假设输入数据服从均匀分布，将数据分到有限数量的桶里，每个桶再分别排序（有可能再使用别的排序算法或是以递归方式继续使用桶排序进行排序）。

> 其实更像是将数轴分段，然后看数所落在哪一段

```js
const bucketSort = arr => {
  let bucketsCount = 10; /* 默认桶的数量 */
  const max = Math.max(...arr); /* 序列最大数字 */
  const min = Math.min(...arr); /* 数列最小数字 */
  const bucketCells = Math.ceil((max - min) / bucketsCount); /* 桶的深度 */
  // const buckets = []; /* 空桶 */

  const buckets = [...Array(bucketsCount)].map(() => []); // 初始化桶，且其子桶为空
  for (let i = 0, len = arr.length; i < len; ++i) {
    // 遍历
    // 利用映射函数将数据分配到各个桶中
    const index = Math.floor((arr[i] - min) / bucketCells); // 向下取整，因为下标从 0 开始
    buckets[index].push(arr[i]); // 推到桶内
  }

  let result = []; /* 真实序列 */
  for (let i = 0, len = buckets[i].length; i < len; ++i) {
    buckets[i].sort((a, b) => a - b); // 对子桶排序，可以使用任何排序算法，这里直接用原生进行原理性说明
    result.push(...buckets[i]);
  }
  return result;
};
```

### 基数排序

基数排序是通过「分配」和「收集」过程来实现排序。

工作原理是将所有待比较数值（正整数）统一为同样的数字长度，数字较短的数前面补零。然后，从最低位开始，依次进行一次排序。这样从最低位排序一直到最高位排序完成以后，数列就变成一个有序序列。

基数排序的方式可以采用 LSD（Least significant digital） 或 MSD（Most significant digital）。

LSD 的排序方式由键值的 最右边（最小位） 开始，而 MSD 则相反，由键值的 最左边（最大位） 开始。MSD 方式适用于位数多的序列。LSD 方式适用于位数少的序列。

LSD 的基数排序方式是我们常见的形式，工作过程如下图：

![LSD 基数排序](https://images2017.cnblogs.com/blog/849589/201710/849589-20171015232453668-1397662527.gif)

```js
const LSDRadixSort = arr => {
  const max = Math.max(...arr); /* 获取最大值 */
  let digit = `${max}`.length; /* 获取最大值位数 */
  let radix = 1; /* 除数 */
  let buckets = []; /* 空桶 */
  while (digit > 0) {
    /* 入桶 */
    for (let i = 0, len = arr.length; i < len; ++i) {
      const index = Math.floor(arr[i] / radix) % 10; // 取相应的位数
      if (!buckets[index]) {
        buckets[index] = [];
      }
      buckets[index].push(arr[i]); /* 往不同桶里添加数据 */
    }
    arr = [];
    /* 出桶 - 收集*/
    for (let i = 0; i < buckets.length; i++) {
      if (buckets[i]) {
        arr = arr.concat(buckets[i]);
      }
    }
    buckets = [];
    radix *= 10;
    digit--;
  }
  return arr;
};
```

MSD 的基数排序方式需要使用**递归**。MSD 的方式由高位数为基底开始进行分配，但在分配之后并不马上合并回一个数组中，而是在每个「桶」中建立「子桶」，将每个「桶」中的数值按照下一数位的值分配到「子桶」中。在进行完最低位数的分配后再合并回单一的数组中。如下图演示：

![MSD 基数排序](http://image.geekaholic.cn/20191127125917.png@0.8)

```js
const MSDRadixSort = (arr, highest) => {
  if (arr.length < 2 || highest < 1) return arr;
  const max = Math.max(...arr);
  let digit = `${max}`.length;
  let buckets;
  let msd = highest || digit; // 指定的最高位，默认为数字的位数
  let result = [];
  buckets = [...Array(10)].map(_ => []); // 初始化
  /* 入桶 */
  for (let i = 0, len = arr.length; i < len; i++) {
    if (`${arr[i]}`.length < digit) buckets[0].push(arr[i]);
    // 没有最高位，直接压入
    else {
      // index 为对应位数的数字
      const index = Math.floor(arr[i] / Math.pow(10, msd - 1)) % 10;
      buckets[index].push(arr[i]);
    }
  }
  /* 递归 */
  for (let i = 0, len = buckets.length; i < len; i++) {
    result = result.concat(
      buckets[i].length >= 2 ? MSDRadixSort(buckets[i], msd - 1) : buckets[i]
    );
  }
  return result;
};
```

### 恶搞的睡眠排序

```js
const list = [3, 4, 5, 8, 9, 7, 1, 3, 4, 3, 6];
const newList = [];
list.forEach(item => {
  setTimeout(function() {
    newList.push(item);
  }, item * 100);
});
```

## 排序算法的选择

- 数据几乎有序：插入排序
- 数据量小，对效率要求不高但代码简单：希尔排序>插入排序>冒泡排序>选择排序
- 数量量大且要求平稳的效率（出现最坏情况复杂度和平均一样）：堆排序
- 数据量大，效率高且要求稳定：归并排序
- 数据量大，且要求最好的平均效率：快速排序 > 堆排序 > 归并排序【虽然堆排序做到了 O(n·log(n)，而快速排序的最差情况是 O(n²)，但是快速排序的绝大部分时间的效率比 O(n·log(n) 还要快】
- 选择排序绝对没用吗：比冒泡交换次数少

---

参考：

- [十大经典排序算法（动图演示） - 一像素 - 博客园](https://www.cnblogs.com/onepixel/p/7674659.html)
- [前端进阶必备 — 手撕排序算法 | 鱼头的 Web 海洋](https://blog.krissarea.com/cs/fe/sorting.html)
- [优雅的 JavaScript 排序算法（ES6） - 掘金](https://juejin.im/post/5ab62ec36fb9a028cf326c49#heading-47)
