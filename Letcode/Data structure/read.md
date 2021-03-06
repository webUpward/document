[TOC]

# 数据结构

## 复杂度分析
> T(n) = O(f(n)),T(n)表示代码执行的时间, n表示数据规模的大小, f(n) 表示每行代码执行的次数总和, 大O表示代码执行时间随数据规模增长的变化趋势
> 最好情况时间复杂度（best case time complexity）、最坏情况时间复杂度（worst case time complexity）、平均情况时间复杂度（average case time complexity）、均摊时间复杂度（amortized time complexity）

## 复杂度量级
> O(1):一般情况下，只要算法中不存在循环语句、递归语句，即使有成千上万行的代码，其时间复杂度也是Ο(1)
> O(logn)、O(nlogn),如果一段代码的时间复杂度是 O(logn)，我们循环执行 n 遍，时间复杂度就是 O(nlogn)了, 归并排序、快速排序的时间复杂度都是 O(nlogn)
> O(m+n)、O(m*n)

## 数组
> 数组（Array）是一种线性表数据结构。它用一组连续的内存空间，来存储一组具有相同类型的数据。数组支持随机访问，根据下标随机访问的时间复杂度为 O(1)。并且写代码的时候要
注意数组越界,否则会被计算机病毒利用代码中的数组越界访问非法地址
> 事先指定数据大小可以省掉很多次内存申请和数据搬移操作。
> 数组为什么下标从0开始: 为了使得cpu每次访问数组元素都减少一次减法运算. 
- 下标最确切的定义应该是:偏移,如果用 a 来表示数组的首地址，a[0]就是偏移为 0 的位置，也就是首地址,如果数组从 1 开始计数，那我们计算数组元素 a[k]的内存地址就会变为

```一维数组寻址
a[k]_address = base_address + (k-1)*type_size
```
```二维数组寻址
 a [ i ][ j ] (i < m,j < n) 
 address = base_address + ( i * n + j) * type_size
```

## 链表如何实现LRU缓存淘汰算法
> 常见的策略有三种：先进先出策略 FIFO（First In，First Out）、最少使用策略 LFU（Least Frequently Used）、最近最少使用策略 LRU（Least Recently Used）
- 单链表,循环链表,双向链表
- 用空间换时间
- 对于执行较慢的程序，可以通过消耗更多的内存（空间换时间）来进行优化；而消耗过多内存的程序，可以通过消耗更多的时间（时间换空间）来降低内存的消耗
- 链表中的每个结点都需要消耗额外的存储空间去存储一份指向下一个结点的指针，所以内存消耗会翻倍。而且，对链表进行频繁的插入、删除操作，还会导致频繁的内存申请和释放，容易造成内存碎片

## Map
- Map的键实际上是跟内存地址绑定的，只要内存地址不一样，就视为两个键。这就解决了同名属性碰撞（clash）的问题
- 和数组相比，链表更适合插入、删除操作频繁的场景，查询的时间复杂度较高

## 判断是否是回文字符串
- 