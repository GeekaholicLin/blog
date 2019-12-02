「有限次操作」的时间复杂度往往是 O(1)。这里的有限次操作，是指不随数据量的增加，操作次数增加。
「for 循环」的时间复杂度往往是 O(n)。
「树的高度」的时间复杂度往往是 O(lg(n))。树的总节点个数是 n，则树的高度是 lg(n)。

递归的时间复杂度怎么分析？

例子一（前 n 项和）：

```js
function sum(n) {
  if (n === 1) return 1;
  return n + sum(n - 1);
}
```

```text
f(1) = 1
f(n) = f(n-1) + 1
     = f(n-2) + 1 + 1
     = f(1) + (n-1)*1
     = n
```

当`n-m=1`时结束，此时`f(n) = = f(1) + (n-1)*1 = n`。

例子二（二分查找）：

```js
function BS(arr, low, high, target) {
  if (low > high) return -1;
  let mid = Math.floor((low + high) / 2);
  if (arr[mid] === target) return mid;
  else if (arr[mid] < target) return BS(arr, mid + 1, high, target);
  return BS(arr, low, mid - 1, target);
}
```

```text
f(1) = 1
f(n) = f(n/2) + 1
     = f(n/4) + 1 + 1
     = f(n/2^m) + (m * 1)
```

递归结束的时机是当`n/2^m=1`的时候，所以`m=lg(n)`。此时一共有`lg(n)`次`+1`，也就是`f(n) = lg(n)`

例子三（快速排序）：

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

```text
f(1) = 1
f(n) = n + 2*f(n/2)
     = (2^m)*f(n/2^m) + m*n
```

递归结束的时机是`n/2^m=1`，所以`f(n) = n*f(1) + lg(n) * n = n * ((lgn)+1) = nlgn`

---

参考：

- [拜托，面试别再问我时间复杂度了！！！ - shenjian58 的博客 - CSDN 博客](https://blog.csdn.net/shenjian58/article/details/89850706)
