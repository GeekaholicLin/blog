# Top K 问题

指的是在数组中找到前 K 大或前 K 小的数。

## 解法一：快速排序

时间复杂度：O(nlogn)，最坏情况 O(n^2)

优点：直观
缺点：全局排序，做了很多无用功；且涉及遍历排序，需要全部加载，不适用于大数据，因为内存可能不够

```js
function quickSort(arr) {
  if (arr.length < 2) return arr;
  let left = [],
    pivot = arr[0],
    center = [],
    right = [];
  for (let i = 0, len = arr.length; i < len; i++) {
    if (arr[i] < pivot) left.push(arr[i]);
    else if (arr[i] === pivot) center.push(arr[i]);
    else right.push(arr[i]);
  }
  return [...quickSort(left), ...quickSort(center), ...quickSort(right)];
}
function topK(arr, k) {
  return quickSort(arr).slice(-k);
}
var genRandomNums = () => {
  return [...Array(10)].map(_ => Math.floor(Math.random() * 100));
};
var randomNums = genRandomNums();
topK(randomNums, 5);
```

## 解法二：局部排序

只冒泡 k 次即可找到。

时间复杂度：O(nk)

```js
function topK(arr, k) {
  for (let i = 0; i < k; i++) {
    for (let j = 0, len = arr.length; j < len - i; j++) {
      if (arr[j] > arr[j + 1]) {
        [arr[j], arr[j + 1]] = [arr[j + 1], arr[j]];
      }
    }
  }
  return arr.slice(-k);
}
var genRandomNums = () => {
  return [...Array(10)].map(_ => Math.floor(Math.random() * 100));
};
var randomNums = genRandomNums();
topK(randomNums, 5);
```

优点：不需要全局排序
缺点：涉及遍历排序，需要全部加载，不适用于大数据，因为内存可能不够

## 解法三：堆

创建一个「最小堆」结构，初始值数组的前 k 个，堆顶为 k 个数里的最小数。然后遍历数组剩下的值，如果数字小于堆顶的数，则直接丢弃，否则把堆顶的数删除，将遍历的数插入堆中，堆结构进行调整，所以可以保证堆顶的数一定是 k 个数里最小的。遍历完毕后，堆里的 k 个数就是数组中最大的前 K 个数。

找 Top K 大的数构建「最小堆」是因为要保证堆的顶部是最小的，只要存在比堆顶小那么就说明堆中并不是最大的 K 个数，需要交换堆顶以及比较的数

时间复杂度：O(nlogk)，logk 是堆调整的复杂度。

```js
// 堆调整 - 最小堆
function heapify(arr, index, size) {
  let min = index;
  let left = index * 2 + 1;
  let right = index * 2 + 2;
  if (left < size && arr[left] < arr[min]) {
    min = left;
  }
  if (right < size && arr[right] < arr[min]) {
    min = right;
  }
  if (min != index) {
    [arr[index], arr[min]] = [arr[min], arr[index]];
    heapify(arr, min, size); // 递归
  }
}
function topK(arr, k) {
  let size = k;
  let containerArr = arr.slice(0, k);
  // 构建最小堆
  for (let i = Math.floor((size - 1) / 2); i >= 0; i--) {
    heapify(containerArr, i, size);
  }
  for (let i = k, len = arr.length; i < len; i++) {
    // 如果比较数大于最小堆的堆顶数，则「替换」并重新调整最小堆
    if (arr[i] > containerArr[0]) {
      containerArr[0] = arr[i]; // 替换而不是交换
      heapify(containerArr, 0, size); // 从堆顶开始
    }
  }
  return containerArr;
}
var genRandomNums = () => {
  return [...Array(10)].map(_ => Math.floor(Math.random() * 100));
};
var randomNums = genRandomNums();
topK(randomNums, 5);
```

PS: 个人感觉就是对「堆排序」的一个变形。

## 解法四：随机选择算法

主要借助了快速排序中的 partition 思想以及减治法，partition 和 TopK 的关系在于，如果 partition 是第 K 大的数，则其右边区域的数则为 TopK

> 减治法（Reduce&Conquer），把一个大的问题，转化为若干个子问题（Reduce），这些子问题中“只”解决一个，大的问题便随之解决（Conquer）

时间复杂度：O(n)

缺点同快速排序，需要全部加载，不适用于大数据。

```js
function swap(arr, i, j) {
  return ([arr[i], arr[j]] = [arr[j], arr[i]]);
}
// 来自快速排序的 partition
function partition(arr, left, right) {
  let storeIndex = left;
  let pivot = arr[right];
  for (let i = left; i < right; i++) {
    if (arr[i] < pivot) {
      swap(arr, storeIndex, i); // 把小的值放到游标位置
      storeIndex++;
    }
  }
  swap(arr, storeIndex, right); // 结束后，把 pivot 放到准确的位置，也就是游标处
  return storeIndex;
}
function topK(arr, k) {
  if (arr.length <= k) return arr;
  let low = 0;
  let high = arr.length - 1;
  let storeIndex = partition(arr, low, high);
  let leftArr = arr.slice(low, storeIndex);
  let rightArr = arr.slice(storeIndex);
  let nums = rightArr.length; // 后半部分元素个数，包括 storeIndex

  if (nums === k) return rightArr;
  // 右边刚好 K 个
  else if (nums < k) return rightArr.concat(topK(leftArr, k - nums)); // 右边少于 k 个，则从左边只需再找 `k-nums` 个
  return topK(rightArr, k); // 右边大于 K，缩小范围重新找过
}
var genRandomNums = () => {
  return [...Array(10)].map(_ => Math.floor(Math.random() * 100));
};
var randomNums = genRandomNums();
topK(randomNums, 4);
```

## 拓展

在 Top K 问题基础上变形的题目有「第 K 大」的数，解法类似，取值不同而已

- 快速排序：直接取`arr[k-1]`
- 局部排序：取数组倒数第 k 个值
- 堆：完成 topK 后的最小堆堆顶，即`containerArr[0]`
- 随机选择算法：需要将`return`变形一下，而不需要利用返回值，如下代码所示。因为它不是有序的，利用返回值还需要进行排序，所以直接进行变形最好。

```js
function topKth(arr, k) {
  if (arr.length <= k) return arr;
  let low = 0;
  let high = arr.length - 1;
  let storeIndex = partition(arr, low, high);
  let leftArr = arr.slice(low, storeIndex);
  let rightArr = arr.slice(storeIndex);
  let nums = rightArr.length; // 后半部分元素个数，包括 storeIndex
  // 上面部分不变
  // storeIndex 刚好对应第 k 个
  if (nums === k) return arr[storeIndex];
  else if (nums < k) return topKth(leftArr, k - nums); // 不需要数组拼接
  return topKth(rightArr, k); // 右边大于 K，缩小范围重新找过
}
```

## 海量数据的处理思路（了解）

top K 问题很适合采用 `MapReduce` 框架解决，用户只需编写一个 Map 函数和两个 Reduce 函数，然后提交到 Hadoop（采用 Mapchain 和 Reducechain）上即可解决该问题。具体而言，就是首先根据数据值或者把数据 hash(MD5) 后的值按照范围划分到不同的机器上，最好可以让数据划分后（的数据大小能够）一次读入内存，这样不同的机器负责处理不同的数值范围，实际上就是 Map。

得到结果后，各个机器只需拿出各自出现次数最多的前 N 个数据，然后汇总，选出所有的数据中出现次数最多的前 N 个数据，这实际上就是 Reduce 过程。

对于 Map 函数，采用 Hash 算法，将 Hash 值相同的数据交给同一个 Reduce task；对于第一个 Reduce 函数，采用 HashMap 统计出每个词出现的频率，对于第二个 Reduce 函数，统计所有 Reduce task，输出数据中的 top K 即可。

---

参考：

- [拜托，面试别再问我 TopK 了](http://1t.click/bq3j)
- [10 亿个数中找出最大的 10000 个数（top K 问题）](https://blog.csdn.net/will130/article/details/49635429)
