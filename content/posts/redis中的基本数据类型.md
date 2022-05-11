---
title: "Redis中的基本数据类型"
date: 2022-05-11T22:47:24+08:00
lastmod: 2022-05-11T22:47:24+08:00
author: ["Collin"]
categories: 
- Redis
tags: 
- Redis
description: ""
weight: # 输入1可以顶置文章，用来给文章展示排序，不填就默认按时间排序
slug: ""
draft: false # 是否为草稿
comments: true
showToc: true # 显示目录
TocOpen: true # 自动展开目录
hidemeta: false # 是否隐藏文章的元信息，如发布日期、作者等
disableShare: true # 底部不显示分享栏
showbreadcrumbs: true #顶部显示当前路径
cover:
    image: ""
    caption: ""
    alt: ""
    relative: false
---
<a name="Wd8Oq"></a>
# Redis中的基本数据类型
<a name="o7YJZ"></a>
## 基本介绍
redis包含五种键值对值的数据类型，分别是：**String，List，Hash，Sorted Set，Set。<br />**对于键值对中的键，都是String类型的，因为redis是k-v数据库，所以整体而言数据的保存形式就相当于在一张大的哈希表中存储，值存储各种数据类型

同时对于字符串而言，只有字符串的底层数据结构是只有一种SDS，其他的都包含两种数据结构，主要体现为在不同情况下转换底层的数据结构以提高存储效率<br />![image.png](https://cdn.jsdelivr.net/gh/CrazyCollin/image@master/uPic/ECYUH5.png)
<a name="NidzL"></a>

## 键和值的结构组织方式
对于redis而言，为了实现从键到值的快速访问，redis使用了一张全局哈希表来保存所有的键值对<br />哈希表其实就是一个数组，数组中的每一个元素都称为哈希桶，每个哈希桶都保存了键值对数据，哈希桶中的元素保存的实际上是指向具体值的指针

![image.png](https://cdn.jsdelivr.net/gh/CrazyCollin/image@master/uPic/cGqikb.png)
<a name="HmFHw"></a>

## 哈希冲突
对于越来越多的键值对的存储，特定长度的数组在哈希存储时必然会出现哈希冲突，即多个entry同时映射到同一个数组索引上，对于这种现象，**redis采用了链式哈希的办法来将entry同时存储在同一个数组索引上面，即后面加入的entry接在前面的entry上，他们之间以链表的形式进行链接**

**当然越来越多的entry在同一个哈希表索引上时，对于链上靠后的entry的查询时间复杂度就会上升，这时候redis就会重新对所有的entry进行rehash扩容，类比于java中的hashmap，当然hashmap还会对链表进行转红黑树操作，这就有所不同了**

**对于rehash而言，redis默认采用了两个全局哈希表，我们将之命名为哈希表1和哈希表2。**<br />**当一开始插入数据时，默认使用哈希表1，当数据开始增多之后，redis开始进行rehash操作，主要分为三部操作：**

- **为哈希表2分配更大的空间，通常是2倍，以容纳哈希表1rehash后的entry数**
- **把哈希表1中的数据重新映射rehash并拷贝到哈希表2当中**
- **释放哈希表1的空间**

**对于哈希表1而言，作为下一次rehash时中的to区**
<a name="ZkhCk"></a>
### 渐进式rehash
当然对于上面那种rehash而言，假如enrty数量过多的话，rehash的时长必然非常的长，造成无法响应其他请求，对于这种情况redis采用了**渐进式rehash**的策略

**具体渐进式rehash的操作步骤主要是将对enrty的rehash计算分摊至每个请求当中，比如说在处理第一个请求的时候，同时将哈希表1中索引为0的全部entry进行rehash，处理第二个请求的时候将索引为1的全部entry进行rehash，这样就不必一次全部rehash了，同时也能够响应客户端的请求了**

**进行rehash的出发条件就是负载因子，就是哈希表已保存节点数量/哈希表大小**

![image.png](https://cdn.jsdelivr.net/gh/CrazyCollin/image@master/uPic/lpqrxE.png)
<a name="k7CRp"></a>

## 底层数据结构
<a name="shKIz"></a>
### SDS
对于redis中的string数据类型，底层是简单动态字符串，其相比较于c原生的char数组而言，自封装了一个结构体用于承载字符串，具体表现为<br />![image.png](https://cdn.jsdelivr.net/gh/CrazyCollin/image@master/uPic/94At5J.png)<br />对于c原生字符串，SDS的好处就是能够直接获取字符串长度，直接进行预分配，不用担心是否数据溢出，同时其保存的数据是二进制安全的，可以存储包含"\0"的数据，当对字符串进行修改时也是安全的，其会先计算预留的空间，假如不足直接扩容，充足则直接分配追加
<a name="HGA0c"></a>

### 双向链表
redis自封装了一个listNode用于承载链表节点，封装一个list用于存储头尾节点以及相关信息（长度等），但同时链表需要保存前后节点，内存开销变大，同时链表节点不连续，对元素是进行逐个访问的<br />![image.png](https://cdn.jsdelivr.net/gh/CrazyCollin/image@master/uPic/vJvCsK.png)
<a name="s0huQ"></a>

### 压缩列表
压缩列表是一种内存紧凑类型的数据结构，占用一块连续的内存空间，其特点就是**针对不同长度的数据，采用不同的编码，有效的节省内存开销**<br />压缩列表保存较多数据的时候查询效率会降低，同时新增或者修改某个元素的话，压缩列表占用的内存空间就需要重新分配，这可能会造成连锁反应，导致后面的压缩列表一并重新分配内存，造成访问压缩列表性能的下降<br />所以对于压缩列表而言，适合保存数据量或者节点数量不太多的场景，这样进行连锁更新的成本也是可以接受的<br />![image.png](https://cdn.jsdelivr.net/gh/CrazyCollin/image@master/uPic/1Lt284.png)
<a name="czrFV"></a>

### 跳表
有序链表一般只能够逐一查找元素，这样的操作非常缓慢，对于跳表来说，查询效率更高<br />具体来说，**跳表就是在链表的基础上，增加了多级索引，实现一种多层的有序链表，通过多个索引之间的跳转，完成数据的快速定位**

**跳表节点（zskiplistNode）同时保存元素和元素的权重值（score），每一个跳表节点都有一个后向指针，指向前一个节点，目的是为了方便从跳表的尾节点开始访问节点，方便倒序查找**

**跳表是一个带有层级关系的链表，每一层可包含多个节点，每个节点之间通过指针连接起来具体就是通过跳表节点中的level数组，level数组的每一个元素都代表着跳表的一层**
```c
typedef struct zskiplistNode {
    //Zset 对象的元素值
    sds ele;
    //元素权重值
    double score;
    //后向指针
    struct zskiplistNode *backward;
  
    //节点的level数组，保存每层上的前向指针和跨度
    struct zskiplistLevel {
        struct zskiplistNode *forward;
        unsigned long span;
    } level[];
} zskiplistNode;
```
同时对于整个跳表的数据结构，如下
```c
typedef struct zskiplist {
    struct zskiplistNode *header, *tail;
    unsigned long length;
    int level;
} zskiplist;
```
包括了跳表的头尾节点，跳表的长度和跳表的层数<br />![image.png](https://cdn.jsdelivr.net/gh/CrazyCollin/image@master/uPic/4omOSQ.png)
<a name="LGQqy"></a>

#### 跳表查询过程
<a name="ynXYE"></a>
#### 跳表节点层数设置
<a name="rlJyn"></a>
### 整形数组
本质上其实就是一块连续的内存空间，元素数量或数据量不大的情况下用整形数组<br />整形数组还有一种升级策略，具体就是当新加入的元素类型比之前数组中保存的元素类型要大的情况下，比如int64和int32这种情况，元素的类型会进行再转换为高阶类型，这样的好处就是在数据占用量不大的情况下节省内存空间
<a name="jGC0J"></a>
### 数据结构的时间复杂度
![image.png](https://cdn.jsdelivr.net/gh/CrazyCollin/image@master/uPic/Eb4hL6.png)

