---
layout: post
title: Why do B and B+ tree are good for MySQL index
date:   2018-08-30 10:53:06
categories: Database
image: /assets/images/post.jpg
---

### 数据库索引-B树 VS B+树

二叉搜索树为什么不适合作数据库索引：
- 数据量大的时候，数的高度会比较高，查询的时候会比较慢
- 每个节点只存储一个记录，可能导致一次查询有很多次磁盘IO

B树的特点：
- 不是二叉搜索树，而是m叉搜索树，一个节点可以存n个记录，这样大大降低了树的高度。比如一个节点可以存100个记录，则高度为3的数可以存至少100w个记录
- 叶子节点，非叶子节点都存储数据
- 通过中序遍历可以忽的所有节点

B数适合做索引：
- m叉分数，高度可以大大降低
- 每个节点可以存储n个记录，如果将节点大小设置为页4K大小，能够充分的利用预读的特性，极大减少磁盘IO

### 局部性原理(磁盘预读)
- 内存读写快，磁盘读写慢很多，可以说不在一个数量级
- 磁盘预读： 磁盘读写并不是按需读取，而是按页预读(一页数据一般是4K)，一次会读一页的数据，每次加载更多的数据，如果未来要读的数据就在这一页中，可以较少未来的磁盘IO，从之前读取的数据中拿到需要的数据，提高效率
- 局部性原理：软件设计要尽量遵循“数据读取集中”与“使用到一个数据，大概率会使用其附近的数据”，这样磁盘预读能充分提高磁盘IO

B+树的特点：
- 是在B树的基础上进行的升级改造，继承了B树的优点
- 非叶子节点不再存储数据，数据只存在同一层的叶子节点上
- 叶子之间，增加了链表，获取所有节点的时候不再需要中序遍历，链表直接遍历效率提升

B+树比B树更优的特性：
- 范围查找， 定位min和max之后，中间叶子节点就是结果集，不用中序回朔。范围查找在SQL中用的很多，这是B+树比B树的最大优势
- 叶子节点存储实际记录行，记录行相对比较紧密的存储，适合大数据量磁盘存储，非叶子节点存储记录的PK，用于查询加速，适合内存存储
- 非叶子节点不存储实际记录，而只存储记录的KEY，那么在相同内存的情况下，B+数能够存储更多的所有

存储例子：

```
(1)局部性原理，将一个节点的大小设为一页，一页4K，假设一个KEY有8字节，一个节点可以存储500个KEY，即j=500

(2)m叉树，大概m/2<= j <=m，即可以差不多是1000叉树

(3)那么：

一层树：1个节点，1*500个KEY，大小4K

二层树：1000个节点，1000*500=50W个KEY，大小1000*4K=4M

三层树：1000*1000个节点，1000*1000*500=5亿个KEY，大小1000*1000*4K=4G
```

存储大量的数据（5亿），并不需要太高树的深度（高度3），索引也不是太占内存（4G）

B+树对于数据库索引的优点：

1. 很适合磁盘存储，能够充分利用局部性原理，磁盘预读；

2. 树高度可以很低，却能够存储大量数据；

3. 索引本身占用的内存很小；

4. 能够很好的支持单点查询，范围查询，有序性查询；