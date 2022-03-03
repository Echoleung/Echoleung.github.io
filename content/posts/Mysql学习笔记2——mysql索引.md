---
title: "Mysql学习笔记2——mysql索引"
date: 2021-11-1
draft: true
tags: ["MySql"]
categories: [""]
description: ""
summary: ""
---

## 2.1、数据结构及算法基础

### 2.1.1、索引的本质

MySQL 官方对索引的定义为：索引（Index）是帮助 MySQL 高效获取数据的数据结构。提取主干即可得到索引的本质：索引是一种数据结构。

数据库查询是数据库的最主要功能之一。我们都希望查询数据的速度能尽可能的快，因此数据库系统的设计者会从查询算法的角度进行优化。例如[二分查找](http://en.wikipedia.org/wiki/Binary_search_algorithm)（binary search）、[二叉树查找](http://en.wikipedia.org/wiki/Binary_search_tree)（binary tree search）等。但是如果稍微分析一下会发现，每种查找算法都只能应用于特定的数据结构之上，例如二分查找要求被检索数据有序，而二叉树查找只能应用于[二叉查找树](http://en.wikipedia.org/wiki/Binary_search_tree)上，但是数据本身的组织结构不可能完全满足各种数据结构。

所以，在数据之外，数据库系统还维护着满足特定查找算法的数据结构，这些数据结构以某种方式引用（指向）数据，这样就可以在这些数据结构上实现高级查找算法。这种数据结构，就是索引。

![](https://img-blog.csdnimg.cn/20201122103829175.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQyMzcyMDE3,size_16,color_FFFFFF,t_70#pic_center)

假设该数据结构就是二叉查找树，左边是数据表，一共有两列七条记录，最左边的是数据记录的物理地址（注意逻辑上相邻的记录在磁盘上也并不是一定物理相邻的）。想要加快 Col2 的查找，可以维护一个右边所示的二叉查找树，每个节点分别包含索引键值和一个指向对应数据记录物理地址的指针，这样就可以通过二叉查找在 O(log_2n) 的复杂度内获取到相应数据，相比于顺序查找 O(n) 速度更快。

虽然这是一个货真价实的索引，但是实际的数据库系统几乎没有使用二叉查找树或其进化品种[红黑树](http://en.wikipedia.org/wiki/Red-black_tree)（red-black tree）实现。

### 2.1.2、B-Tree

![](https://img-blog.csdnimg.cn/20201122104411794.png#pic_center)

在 B-Tree 中按 key 检索数据的算法非常直观：首先从根节点进行二分查找，如果找到则返回对应节点的 data，否则对相应区间的指针指向的节点递归进行查找，直到找到节点或找到 null 指针，前者查找成功，后者查找失败。B-Tree 上查找算法的伪代码如下：

```
BTree_Search(node, key) {
    if(node == null) return null;
    foreach(node.key)
    {
        if(node.key[i] == key) return node.data[i];
            if(node.key[i] > key) return BTree_Search(point[i]->node);
    }
    return BTree_Search(point[i+1]->node);
}
data = BTree_Search(root, my_key);

```

### 2.1.3、B+Tree

B-Tree 有许多变种，其中最常见的是 B+Tree，例如 MySQL 就普遍使用 B+Tree 实现其索引结构。

与 B-Tree 相比，B+Tree 有以下不同点：

-   每个节点的指针上限为 2d 而不是 2d+1
-   内节点不存储 data，只存储 key
-   叶子节点不存储指针

![](https://img-blog.csdnimg.cn/20201122104441614.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQyMzcyMDE3,size_16,color_FFFFFF,t_70#pic_center)

### 2.1.4、带有顺序访问指针的 B+Tree

数据库系统或文件系统中使用的 B+Tree 结构都在经典 B+Tree 的基础上进行了优化，增加了顺序访问指针。

![](https://img-blog.csdnimg.cn/20201122104457233.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQyMzcyMDE3,size_16,color_FFFFFF,t_70#pic_center)

在 B+Tree 的每个叶子节点增加一个指向相邻叶子节点的指针，就形成了带有顺序访问指针的 B+Tree。做这个优化的目的是为了提高区间访问的性能，例如如果要查询 select \* from user where age > 18，当找到 18 后，只需顺着节点和指针顺序遍历就可以一次性访问到所有数据节点，极大提到了区间查询效率。

## 2.2、为什么使用 B-Tree（B+Tree）

红黑树等数据结构也可以用来实现索引，但是文件系统及数据库系统普遍采用 B-/+Tree 作为索引结构，这一节将结合计算机组成原理相关知识讨论 B-/+Tree 作为索引的理论基础。

一般来说，索引本身也很大，不可能全部存储在内存中，因此索引往往以索引文件（MyISAM 和 InnoDB 都是）的形式存储的磁盘上。索引的查找过程中就要产生磁盘 I/O 消耗，相对于内存存取，I/O 存取的消耗要高几个数量级，所以评价一个数据结构作为索引的优劣最重要的指标就是在查找过程中磁盘 I/O 操作次数的渐进复杂度。所以，索引的结构组织要尽量减少查找过程中磁盘 I/O 的存取次数。

-   下面先介绍内存和磁盘存取原理，然后再结合这些原理分析 B-/+Tree 作为索引的效率。

> 主存存取原理（快）

目前计算机使用的主存基本都是随机读写存储器（RAM），现代 RAM 的结构和存取原理比较复杂，下面就简单介绍一下。

![](https://img-blog.csdnimg.cn/20201122104551378.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQyMzcyMDE3,size_16,color_FFFFFF,t_70#pic_center)

从抽象角度看，主存是一系列的存储单元组成的矩阵，每个存储单元存储固定大小的数据。每个存储单元有唯一的地址，现代主存的编址规则比较复杂，这里将其简化成一个二维地址：通过一个行地址和一个列地址可以唯一定位到一个存储单元。

主存的存取过程如下：

-   当系统需要读取主存时，则将地址信号放到地址总线上传给主存，主存读到地址信号后，解析信号并定位到指定存储单元，然后将此存储单元数据放到数据总线上，供其它部件读取。
-   写主存的过程类似，系统将要写入单元地址和数据分别放在地址总线和数据总线上，主存读取两个总线的内容，做相应的写操作。
-   主存存取的时间仅与存取次数呈线性关系，因为不存在机械操作，两次存取的数据的 “距离” 不会对时间有任何影响，例如，先取 A0 再取 A1 和先取 A0 再取 D3 的时间消耗是一样的。

> 磁盘存取原理（慢）

索引一般以文件形式存储在磁盘上，索引检索需要磁盘 I/O 操作。与主存不同，磁盘 I/O 存在机械运动耗费，因此磁盘 I/O 的时间消耗是巨大的。

![](https://img-blog.csdnimg.cn/20201122104637857.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQyMzcyMDE3,size_16,color_FFFFFF,t_70#pic_center)

当需要从磁盘读取数据时，系统会将数据逻辑地址传给磁盘，磁盘的控制电路按照寻址逻辑将逻辑地址翻译成物理地址，即确定要读的数据在哪个磁道，哪个扇区。为了读取这个扇区的数据，需要将磁头放到这个扇区上方，为了实现这一点，磁头需要移动对准相应磁道，这个过程叫做寻道，所耗费时间叫做寻道时间，然后磁盘旋转将目标扇区旋转到磁头下，这个过程耗费的时间叫做旋转时间。

**局部性原理与磁盘预读**

由于存储介质的特性，磁盘本身存取就比主存慢很多，再加上机械运动耗费，磁盘的存取速度往往是主存的几百分分之一，因此为了提高效率，要尽量减少磁盘 I/O。为了达到这个目的，磁盘往往不是严格按需读取，而是每次都会预读，即使只需要一个字节，磁盘也会从这个位置开始，顺序向后读取一定长度的数据放入内存。

这样做的理论依据是计算机科学中著名的局部性原理：当一个数据被用到时，其附近的数据也通常会马上被使用。

由于磁盘顺序读取的效率很高（不需要寻道时间，只需很少的旋转时间），因此对于具有局部性的程序来说，预读可以提高 I/O 效率。

预读的长度一般为页（page）的整倍数。页是计算机管理存储器的逻辑块，硬件及操作系统往往将主存和磁盘存储区分割为连续的大小相等的块，每个存储块称为一页（在许多操作系统中，页得大小通常为 4k），主存和磁盘以页为单位交换数据。当程序要读取的数据不在主存中时，会触发一个缺页异常，此时系统会向磁盘发出读盘信号，磁盘会找到数据的起始位置并向后连续读取一页或几页载入内存中，然后异常返回，程序继续运行。

> B-/+Tree 索引的性能分析

一般使用磁盘 I/O 次数评价索引结构的优劣。

先从 B-Tree 分析，根据 B-Tree 的定义，可知检索一次最多需要访问 h 个节点。数据库系统的设计者巧妙利用了磁盘预读原理，将一个节点的大小（16KB）设为等于一个页，这样每个节点只需要一次 I/O 就可以完全载入内存。为了达到这个目的，在实际实现 B-Tree 还需要使用如下技巧：

-   每次新建节点时，直接申请一个页的空间，这样就保证一个节点物理上也存储在一个页里，加之计算机存储分配都是按页对齐的，就实现了一个 node 只需一次 I/O。
-   B-Tree 中一次检索最多需要 h-1 次 I/O（根节点常驻内存），因此渐进复杂度为 (O(h)=O(logdN))。一般实际应用中，出度 d 是非常大的数字，通常超过 100，因此 h 非常小（通常不超过 3）。

内节点出度 d 也可以理解为一个结点能存储的数据（包含一个 key 域、数据域、指针域）组数

计算公式：d\_{max}=floor(pagesize / (keysize + datasize + pointsize))

-   floor 表示向下取整
-   pagesize 大小固定 16KB，所以 (keysize + datasize + pointsize) 越小，d 也将越大，一个结点内能够储存的数据组数也就越多。  
    综上所述，用 B-Tree 作为索引结构效率是非常高的。

而红黑树这种结构，h 明显要深的多（每次需要将磁盘上的数据读入内存才能进行比较，越深就说明需要读取的结点数越多）。由于逻辑上很近的节点（父子）物理上可能很远，无法利用局部性，所以红黑树的 I/O 渐进复杂度也为 O(h)，效率明显比 B-Tree 差很多。

B+Tree 更适合外存索引，原因和内节点出度 d 有关。从上面分析可以看到，d 越大索引的性能越好，而出度的上限取决于节点内 key 和 data 的大小，由于 B+Tree 内节点去掉了 data 域，因此可以拥有更大的出度，拥有更好的性能。

## 2.3、MySQL 索引实现

在 MySQL 中，索引属于存储引擎级别的概念，不同存储引擎对索引的实现方式是不同的，主要讨论 MyISAM 和 InnoDB 两个存储引擎的索引实现方式。

> MyISAM 索引实现（非聚集）

MyISAM 引擎使用 B+Tree 作为索引结构，叶节点的 data 域存放的是数据记录的地址。

![](https://img-blog.csdnimg.cn/20201122105345170.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQyMzcyMDE3,size_16,color_FFFFFF,t_70#pic_center)

![](https://img-blog.csdnimg.cn/20201122105345141.png#pic_center)

-   .frm 文件—表结构的定义
-   .MYD 文件—MyISAM 的简写 + Data 存储数据
-   .MYI 文件—MyISAM 的简写 + Index 存储索引

MyISAM 中索引检索的算法为首先按照 B+Tree 搜索算法搜索索引，如果指定的 Key 存在，则取出其 data 域的值，然后以 data 域的值为地址，读取相应数据记录。

MyISAM 的数据和索引是存储在不同的文件中，所以索引方式也叫做 “非聚集” 的。

> InnoDB 索引实现（聚集）

虽然 InnoDB 也使用 B+Tree 作为索引结构，但具体实现方式却与 MyISAM 截然不同。

1、InnoDB 的数据文件本身就是索引文件。从上文知道，MyISAM 索引文件和数据文件是分离的，索引文件仅保存数据记录的地址。而在 InnoDB 中，表数据文件本身就是按 B+Tree 组织的一个索引结构，这棵树的叶节点 data 域保存了完整的数据记录。这个索引的 key 是数据表的主键，因此 InnoDB 表数据文件本身就是主索引。

![](https://img-blog.csdnimg.cn/20201122105508827.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQyMzcyMDE3,size_16,color_FFFFFF,t_70#pic_center)

叶节点包含了完整的数据记录，这种索引叫做聚集索引。因为 InnoDB 的数据文件本身要按主键聚集，所以 InnoDB 要求表必须有主键（MyISAM 可以没有），如果没有显式指定，则 MySQL 系统会自动选择一个可以唯一标识数据记录的列作为主键，如果不存在这种列，则 MySQL 自动为 InnoDB 表生成一个隐含字段作为主键，这个字段长度为 6 个字节，类型为长整形。

-   为什么 InnoDB 表必须有主键，并且推荐使用整型的自增主键？

解答：假设使用 UUID 作为主键（解释整型）

1.  使用索引进行查询数据的时候，需要对主键进行比较，整型的比较速度比字符串快。
2.  UUID 的位数比较长，也就是上面分析过的 keysize 比整型占用得多，其结果就是内节点出度 d 降低，查询性能降低！

自增和指针的合作：（解释自增）

1.  方便范围查找，例如：select \* from user where col > 18;
2.  如果不是递增的插入元素，那么插入一个满结点当中就有可能会造成没有空间存储，分裂 + 数平衡引发额外的开销。

## 2.4、联合索引

![](https://img-blog.csdnimg.cn/20201122105659851.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQyMzcyMDE3,size_16,color_FFFFFF,t_70#pic_center)

按照每个字段出现的先后顺序组成联合索引放到一个节点中存储，排序规则也按字段的大小顺序。

## 2.5、InnoDB 的主键选择与插入优化

在使用 InnoDB 存储引擎时，如果没有特别的需要，请永远使用一个与业务无关的自增字段作为主键。

InnoDB 使用聚集索引，数据记录本身被存于主索引（一颗 B+Tree）的叶子节点上。这就要求同一个叶子节点内（大小为一个内存页或磁盘页）的各条数据记录按主键顺序存放，因此每当有一条新的记录插入时，MySQL 会根据其主键将其插入适当的节点和位置，如果页面达到装载因子（InnoDB 默认为 15/16），则开辟一个新的页（节点）。

如果表使用自增主键，那么每次插入新的记录，记录就会顺序添加到当前索引节点的后续位置，当一页写满，就会自动开辟一个新的页。如下图所示：

![](https://img-blog.csdnimg.cn/20201122105733200.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQyMzcyMDE3,size_16,color_FFFFFF,t_70#pic_center)

这样就会形成一个紧凑的索引结构，近似顺序填满。由于每次插入时也不需要移动已有数据，因此效率很高，也不会增加很多开销在维护索引上。

如果使用非自增主键（如果身份证号或学号等），由于每次插入主键的值近似于随机，因此每次新纪录都要被插到现有索引页得中间某个位置：

![](https://img-blog.csdnimg.cn/20201122105750836.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQyMzcyMDE3,size_16,color_FFFFFF,t_70#pic_center)

此时 MySQL 不得不为了将新记录插到合适位置而移动数据，甚至目标页面可能已经被回写到磁盘上而从缓存中清掉，此时又要从磁盘上读回来，这增加了很多开销，同时频繁的移动、分页操作造成了大量的碎片，得到了不够紧凑的索引结构，后续不得不通过 OPTIMIZE TABLE 来重建表并优化填充页面。
