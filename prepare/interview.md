- 什么是闭包：函数与对其状态即词法环境（lexical environment）的引用共同构成闭包（closure），以及闭包的应用场景
- 浮点数运算精度问题
- 背包问题，二叉树算法，LRU 和 LFU
- React 的 diff 算法
- redux 与 react-redux 等
- 对合成层的理解
- 活动倒计时的精确
- 正则表达式
- ES5 和 ES6 继承的区别
- axios 拦截器
- HTML5 原生拖放
- scss 的基础和进阶

---

哈希表的原理并不复杂，简而言之就是根据 Key 来计算出存储位置（哈希函数），然后将数据放入该空间，查询时同样根据 Key 计算出存储位置后直接将相应的值取出。

构造哈希函数有三个要点：（1）运算过程要尽量简单高效，以提高哈希表的插入和检索效率；（2）哈希函数应该具有较好的散列型，以降低哈希冲突的概率；（3）哈希函数应具有较大的压缩性，以节省内存

通用的哈希函数：

- 除留取余法：最常用。H(x) = x % p，其中 x 为存储的值，p 为不超过容量的最大质数
- 直接定址法：`H(x) = a * x + b`，a,b 取值自定
- 折叠法：
- 平方取中法：计算数据的平方，然后从平方数中选出中间几位来作为存储的地址
- 数字分析法：完全通过观察数据规律来确定相应 key 的方法。比如`1125699 1123399 1158299`
  通过观察这组数据，发现开头和结尾都一样，那么可以选择中间三位来确定 key 值

## 冲突解决：

在实际应用中，冲突是无法完全避免的。

- 链表地址法：将有冲突的数据放在一个链表里，当查询时会根据 key 查到链表的第一个节点，然后遍历整个链表，找到相应的值。
- 开放定址法：当发生地址冲突时，按照某种方法继续探测哈希表中的其他存储单元，直到找到空位置为止。最具代表性的一种是**线性探测法**，当发现冲突的时候，依次往后遍历，直到找到空的位置。但会导致冲突累计，解决一个冲突的同时会占据别的 key 的位置，又造成了新的冲突
- 二次方探测法：
-

---

- [前端面试与进阶指南](https://www.cxymsg.com/)
- [awesome-coding-js](http://www.conardli.top/docs/)
- [心谭博客](https://xin-tan.com/)
- [前端进阶之道](https://yuchengkai.cn/home/)
- [高级前端进阶博文 | 木易杨前端进阶](https://muyiy.cn/blog/)

- [azl397985856/leetcode](https://github.com/azl397985856/leetcode)
- [FEInterviewBox/剑指 offer](https://github.com/14glwu/FEInterviewBox/tree/master/%E5%89%91%E6%8C%87offer)
- [javascript-algorithms](https://github.com/trekhleb/javascript-algorithms/blob/master/README.zh-CN.md)

- [前端面试总结（at, md） - 掘金](https://juejin.im/post/5a3134bf6fb9a0452405d507)
- [面试官：你有 m 个鸡蛋，如何用最少的次数测出鸡蛋会在哪一层碎？ - 掘金](https://juejin.im/post/5d9ede57518825358b221349)
- [拜托，面试官别问我「布隆」了 - 掘金](https://juejin.im/post/5c959ff8e51d45509e2ccf84)
- [中高级前端大厂面试秘籍，为你保驾护航金三银四，直通大厂（上）](https://juejin.im/post/5c64d15d6fb9a049d37f9c20)
- [2 万 5 千字大厂面经 | 掘金技术征文 - 掘金](https://juejin.im/post/5ba34e54e51d450e5162789b)
- [youngwind/blog: 梁少峰的个人博客](https://github.com/youngwind/blog)

- [🚵 前端性能优化之旅 | 前端性能优化](https://alienzhou.github.io/fe-performance-journey/)
- https://blog.csdn.net/shenjian58/article/details/89850920
