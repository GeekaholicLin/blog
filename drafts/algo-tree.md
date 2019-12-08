# 树的遍历算法

## 存储表示方法

1. 双亲表示法：

```js
// JavaScript
[
  { value: "node1", parent: -1 },
  { value: "node2", parent: 0 },
  { value: "node3", parent: 0 },
  { value: "node4", parent: 0 },
  { value: "node5", parent: 1 },
  { value: "node6", parent: 2 },
  { value: "node7", parent: 2 }
];
```

2. 孩子链式表示法：

```js
[
  {
    value: "node1",
    children: [{ value: "node2", children: null }]
  }
];
```

3. 左右结构表示法：

```js
[
  {
    value: "node1",
    left: { value: "node2", left: null, right: null },
    right: { value: "node3" }
  }
];
```

## 知识储备

### 概念和关系

- 根节点是树最顶层节点
- 边是两个节点之间的连接
- 子节点是具有父节点的节点
- 父节点是与子节点有连接的节点
- 叶子节点是树中没有子节点的节点（树得末端）
- 树的高度（height）是到叶子节点（树末端）的长度
- 节点的深度（depth）是本身到根节点的长度
- 在二叉树的第 i（根节点为 1 层）层上最多有 2^(i-1) 个节点（i>=1）
- 高度为 k 的二叉树，最多有`2^k -1`个节点
- 叶子节点的度为 0，度为二的节点是指有两个子节点的节点
- 树中结点数 = 所有结点的度之和 + 1（这个加一是因为还要计算根节点，因为它不是任何节点的子节点）

### 种类

- 二叉树是一种树形数据结构，它的每个节点最多有两个子节点
- 满二叉树是除叶子节点外的其他节点都有两个子节点，且叶子结点只能出现在最下一层。
- 完全二叉树是一棵二叉树至多只有最下面的两层上的结点的度数可以小于 2，并且最下层上的结点都集中在该层最左边的若干位置上（特点就是从左到右依次填充）
- 二叉排序树是每个节点的值均大于其左子树上所有节点的值，小于其右子树上所有节点的值，对二叉排序树进行中序遍历得到一个有序序列
- 平衡二叉树（Balanced BinaryTree）又被称为 AVL 树（有别于 AVL 算法），它是一棵空树或它的左右两个子树的高度差的绝对值不超过 1，并且左右两个子树都是一棵平衡二叉树。
- 堆是一种特殊的完全二叉树，所有父节点比子节点大的完全二叉树是最大堆，所有父节点比子节点小的完全二叉树是最小堆

### 二叉树遍历

#### 广度遍历

```js
// 一个数组保存值，一个数组作为队列保存要打印的节点顺序
function PrintFromTopToBottom(root) {
  if (!root) return [];
  let treeQueue = [];
  let result = [];
  treeQueue.push(root);
  let node;
  while (treeQueue.length) {
    node = treeQueue.shift(); // 队首节点
    result.push(node.val);
    if (node.left) {
      treeQueue.push(node.left);
    }
    if (node.right) {
      treeQueue.push(node.right);
    }
  }
  return result;
}
```

#### 前中后遍历

```js
// 前序-递归形式
function preOrder(root) {
  if (root) {
    console.log(root.val);
    preOrder(root.left);
    preOrder(root.right);
  }
}
// 前序-非递归形式，借助栈先进后出的特性，因为它要不断地递归（只要有左就先左）
function preOrderUnRecur(root) {
  if (!root) return;
  var stack = [];
  stack.push(root);
  while (stack.length) {
    root = stack.pop();
    console.log(node.val);
    if (root.right) stack.push(root.right);
    if (root.left) stack.push(root.left);
  }
}
// 中序-递归形式
function InOrder(root) {
  if (root) {
    InOrder(root.left);
    console.log(root.val);
    InOrder(root.right);
  }
}
// 如果有根节点根节点入栈，然后更新 node 指向左子树
// 否则弹出栈顶元素，此时栈顶元素为子树的父节点
function InOrderUnRecur(root) {
  if (!root) return;
  var stack = [];
  stack.push(root);
  while (stack.length || root) {
    if (root) {
      stack.push(root);
      root = root.left;
    } else {
      root = stack.pop();
      console.log(root.val);
      root = root.right;
    }
  }
}
// 后序-递归形式
function postOrder(root) {
  if (root) {
    postOrder(root.left);
    postOrder(root.right);
    console.log(root.val);
  }
}
// 借助两个栈
function postOrderUnRecur(root) {
  if (!root) return;
  var stack1 = [];
  var stack2 = [];
  stack1.push(root);
  // 全部压入 stack2
  while (stack1.length) {
    root = stack1.pop();
    stack2.push(root);
    if (root.left) stack1.push(root.left);
    if (root.right) stack1.push(root.right);
  }
  while (stack2.length) {
    console.log(s2.pop().val);
  }
}
```

### 二叉排序树（二叉搜索树，Binary Search Tree）和堆的关系

在二叉排序树中，每个节点的值均大于其左子树上所有节点的值，小于其右子树上所有节点的值，对二叉排序树进行中序遍历得到一个有序序列。所以，二叉排序树是节点之间满足一定次序关系的二叉树。

堆是一个完全二叉树，并且每个节点的值都大于或等于其左右孩子节点的值（这里的讨论以大根堆为例），所以，堆是节点之间满足一定次序关系的完全二叉树。

具有 n 个节点的二叉排序树，其深度取决于给定集合的初始排列顺序，最好情况下其深度为 log n（表示以 2 为底的对数），最坏情况下其深度为 n；具有 n 个节点的堆，其深度即为堆所对应的完全二叉树的深度 log n 。

在二叉排序树中，某节点的右孩子节点的值一定大于该节点的左孩子节点的值；在堆中却不一定，堆只是限定了某节点的值大于（或小于）其左右孩子节点的值，但没有限定左右孩子节点之间的大小关系。

二叉排序树是为了实现动态查找而设计的数据结构，它是面向查找操作的，在二叉排序树中查找一个节点的平均时间复杂度是 O(log n)；

堆是为了实现排序而设计的一种数据结构，它不是面向查找操作的，因而在堆中查找一个节点需要进行遍历，其**查找的**平均时间复杂度是 O(n)

#### 二叉排序树构建、查找和删除
