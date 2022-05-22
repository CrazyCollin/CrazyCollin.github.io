---
title: "Deep Dive Into Message Chunking in Pulsar"
date: 2022-05-22T18:08:54+08:00
lastmod: 2022-05-22T18:08:54+08:00
author: ["Collin"]
categories: 
- 消息队列
tags: 
- 消息队列
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
<a name="EdAqi"></a>
# 深入理解Pulsar的大消息拆分
此篇文章是杨子棵导师关于大消息拆分的博客的个人自行翻译理解部分，原文请见：[https://streamnative.io/blog/engineering/2022-05-10-deep-dive-into-message-chunking-in-pulsar/](https://streamnative.io/blog/engineering/2022-05-10-deep-dive-into-message-chunking-in-pulsar/)
<a name="KZMFV"></a>
## 消息拆分的原理
<a name="BYB4O"></a>
### 单生产者发送分块消息至单topic
![image.png](https://cdn.jsdelivr.net/gh/CrazyCollin/image@master/uPic/EMIs8m.png)<br />当消息拆分选项启动的时候，假如生产者收到从用户收到大消息并且这个大消息大小大于broker传来的maxMessageSize的时候，生产者会把这个消息进行拆分。如图所示，消息M被拆分为：M-C1，M-C2，M-C3。生产者保证每个分块的消息大小不超过maxMessageSize，并且按序将分块消息发出去。<br />消费者在全部分块到达之前首先将这些分块全部缓存起来，然后消费者会将分块拼接起来，消费消息，并把它移交给应用。
<a name="ktQ7H"></a>

### 单生产者混合发送普通消息和分块消息至单topic
![image.png](https://cdn.jsdelivr.net/gh/CrazyCollin/image@master/uPic/WG1PGQ.png)<br />如图所示，消息M1被分为三个普通的消息：M1-C1，M1-C2和M1-C3。消息M2同样被分为两个普通的消息：M2-C1和M2-C2。在topic中同样有两个不是分块的消息：M3和M4.<br />broker将这些消息都存储在topic中，并且按照同样的顺序向消费者分发。消费者能够判断到来的消息是否为分块消息。在接收到分块消息之后，消费者在接收所有的分块信息之前会在内存中将分块信息缓存起来。然后消费者把这些分块信息拼接成一条信息，大消息的信息负载也被移交给应用处理。
<a name="fc30l"></a>

### 多生产者并发发送分块消息至单topic
Pulsar允许多个生产者向单个topic中同时发送消息。broker将来自不同生产者的分块信息存储在同一个topic中。当多个生产者同时向同一个topic发送分块消息的时候，broker将来自不同生产者的分块消息交织存储在同一个topic中。这些分块仍然是有顺序的，只不过可能在topic中不是严格连续的。这给消费者端带来了一定的内存压力，因为消费者需要把每条大消息的分块消息分开来缓存。<br />![image.png](https://cdn.jsdelivr.net/gh/CrazyCollin/image@master/uPic/3N4C8s.png)<br />这张图片就是一个很好的例子。生产者1
> Note：在所有讨论的例子中，只有一个消费者同时从一个topic中消费信息。简单来说，就是对于消费者的订阅类型必须是独占或者是灾备型的。目前Pulsar并不支持共享型或者是key共享型的订阅类型。请参见[Large Message Size Handling in Pulsar: Chunking for Shared Subscription](https://github.com/apache/pulsar/issues/7645)这个issue以讨论这个特性

<a name="xNB5u"></a>
## 如何启用消息分块
消息分块默认是关闭的。你如果想使用这项特性那就需要在创建生产者的时候就启动message chunking选项。<br />这里有几个实用消息分块的限制：

- topic必须是持久化topic
- 在生产者中消息分块不能同时和批量处理选项同时开启
- 订阅类型必须是独占型或者是灾备型的

下面是一个怎么在创建生产者时开启消息分块的示例。
```java
Producer<byte[]> producer = pulsarClient.newProducer()
       .topic(topic)
       .enableChunking(true)
       .enableBatching(false)
       .create();
```
然后你就可以和下面的一样用这个生产者来发送负载大容量信息的消息了：
```java
byte[] data = new byte[10 * 1024 * 1024];
producer.send(data);
```
你同样可以像下面的一样创建一个消费者来消费和确认负载大容量信息的消息：
```java
Consumer<byte[]> consumer = pulsarClient.newConsumer()
       .topic(topic)
       .subscriptionName("test")
       .subscribe();

Message<byte[]> msg = consumer.receive();
consumer.acknowledge(msg);
```
<a name="EfI8I"></a>
## 如何实现消息分块
这个部分深入消息分块的实现。首先让我们看一下和分块消息相关的结构和属性。然后我们会了解产生和消费分块消息的整个过程。<br />从下面展示的来看，在消息元数据中有四个属性和分块消息相关。生产者和消费者都共用这些属性
```java
optional string uuid;
optional int32 num_chunks_from_msg;
optional int32 total_chunk_msg_size; 
optional int32 chunk_id;
```

1. **uuid**用来唯一的标识整个chunk message。在整个chunk message中的每一个分块中的uuid都是相等的。我们可以用这种方法了解一条消息究竟是属于哪一条chunk message。
1. **num_chunks_from_msg**是表示chunk message中的分块数量。整个chunk message中的每个分块的**num_chunks_from_msg**都是一样的。
1. **total_chunk_msg_size**表示整个chunk message负载大小。整个chunk message中的每个分块的**total_chunk_msg_size**都是一样的。
1. **chunk_id**用来标记当前消息的特殊部分（？）。
<a name="qzWKE"></a>
### 拆分chunked messages
让我们先来看看生产者端的消息分块功能实现。在客户端和broker成功建立连接之后，broker将消息最大值maxMessageSize通过序列化proto命令CommandConnected传输给客户端。这是一个可选属性，默认值为5MB。<br />当生产者发送消息的时候，它会当前消息的负载大小是否已经超过之前broker传给自己的maxMessageSize。加入消息已经超过大小限制，那么大消息就会被切分。比如一条承载信息大小为12mb的信息就会被切分为以下三块。<br />![image.png](https://cdn.jsdelivr.net/gh/CrazyCollin/image@master/uPic/L2R7hm.png)
<a name="euwAc"></a>

### 推送chunked messages
当大消息被分块，生产者将每一个分块当作普通的消息发给broker。每一个分块就像普通的消息一样，同样受到了broker的流量控制和客户端的内存限制。在生产者配置中有一个maxPendingMessages参数来限制了生产者能够同时发送消息的最大数量。每一个分块在maxPendingMessages中都被单独的计数。这意味着发送一个包括三个分块的大消息会占用生产者中等待发送消息的位置。（？）<br />每当发送一个分块，就有一个单独的OpSendMsg为了每个分块而创造出来。每个OpSendMsg共享着同一个用来返回给用户的分块message ID的ChunkMessageCtx上下文。每个分块同样在自己的消息元数据中共享同一个uuid。<br />生产者按顺序将每个分块发给broker，并且broker按序接收。这保证了当最后一个分块的推送被确认后整个chunk message的成功发送。这同样保证了chunk message的所有分块消息在topic中按序存放。消费者依赖这个顺序来保证消费分块时的有序。<br />对于一个有着分区的topic，生产者直接将大消息中的所有分块推送到同一个分区当中。
<a name="mAKOo"></a>
### chunked message ID
在pulsar中，当一个普通消息被推送或者被消费掉时，它的message ID将被返回它的用户。对于chunk message而言，pulsar是如何将message ID返回给它的用户的呢？
<a name="YKU0B"></a>
#### 在Pulsar2.10.0版本之前的chunk message ID
在pulsar2.10.0之前，生产者或者消费者只返回最后一个分块的message ID作为整个chunk message的message ID。这个实现在有时候会引发问题。举个例子，当我们用这个message ID来寻找的时候，消费者会从最后一个分块的位置中开始消费。那么这个消费者就会错误的认为前面的分块已经丢失并且会选择跳过当前的消息。如果我们广泛的搜索，那么消费者很有可能跳过某些消息，这也会导致一些不可预期的事情发生。<br />![image.png](https://cdn.jsdelivr.net/gh/CrazyCollin/image@master/uPic/ZNJ5UD.png)<br />从图中我们可以知道，消费者将最后一个分块的消息作为chunk message的message ID返回给用户。如果用户用这个message ID来做一个泛查询，在broker的视角来看，消费者是在找M1-C3.根据泛查询的语意来看，在查询后第一个被消费者消费的消息应该是M1。但实际上，第一个进入消费者的消息是M1-C3。消费者会发现它并没有收到chunk message前面的分块部分，并且不能够继续接受之前的分块，所以它会直接抛弃M1。因此，实际上被消费的第一个消息是M2.这是一个预料之外的事情。
<a name="ouAAD"></a>

#### 在Pulsar2.10.0版本及之后的chunk message ID
为了解决这个问题，pulsar在2.10.0之后引入了chunk message ID。PIP107详细阐述了这个特性。chunk message ID和原来的行为保存一致，以实现与原始逻辑的兼容。chunk message ID包含两个普通的message ID：第一个分块和最后一个分块的message ID。生产者和消费者都用chunk message上下文来生成chunk message ID。<br />![image.png](https://cdn.jsdelivr.net/gh/CrazyCollin/image@master/uPic/HXtvMz.png)<br />从图中可见，当生产者收到每一个分块的推送确认的时候获得第一个分块和最后一个分块的message ID。生产者将其缓存在chunk message上下文当中，并且在收到最后一个分块的message ID后生成chunk message ID。对于消费者来说也是同样一个过程。区别就是消费者通过接收消息来获取message ID。<br />chunk message ID功能不仅解决了seeking情况下导致的问题，而且还允许你获取关于chunk message更多的信息。
<a name="kJf7s"></a>

### 组合chunked messages
消费者需要在消息交付给应用之前将所有的分块重新整合成原始消息。消费者使用chunk message上下文来缓存所有的分块数据，比如说负载数据，元数据和消息ID这种。当处理分块消息的时候，消费者会假定分块被按序接收并且在分块乱序的情况下将所有的分块丢弃。<br />![image.png](https://cdn.jsdelivr.net/gh/CrazyCollin/image@master/uPic/5Dtc4E.png)<br />假设我们推送了两个大消息，“abc”和“de”。它们在topic中等待被消费。mexMessageSize被设置为1byte的大小，这导致了每个分块负载信息也比较小。<br />当消费者消费第一个消息的时候，它发现这是一个chunk message（主要是从元数据中取chunk sizes），紧接着创建一个ChunkedMessageCtx上下文。uuid唯一标识chunk message，以便我们只当将分块置于哪一个chunk message上下文中。上下文中的lastChunkedMessageId表示最后收到的分块ID。每当消费者收到新块时，这个值都将被更新。上下文中的负载信息也就是整个chunk message负载信息的当前缓存值。这个在消费者接收分块时会一直保持增长（像下图所示）。<br />![image.png](https://cdn.jsdelivr.net/gh/CrazyCollin/image@master/uPic/5qekrd.png)<br />像上图所示一样，当消费者全部收到了uuid为1的chunk message的所有分块时，它就能够用chunked message上下文生成原始消息，也就是生产者端的大消息。这个完整的消息随后被移交给应用并且消费者释放上下文。值得注意的是在这个阶段因为我们已经收到一个uuid为2的chunk message的分块，第二个chunked message上下文也因此在消费者中创建。<br />![image.png](https://cdn.jsdelivr.net/gh/CrazyCollin/image@master/uPic/cyBQwk.png)<br />就像之前的例子一样，当消费者收到了uuid为2的全部分块，它就会从chunked message上下文中生成一条新消息，也就是大消息，并把这个完整的消息移交给应用。
<a name="olxfd"></a>

## chunked messages的最佳实践
在这个部分，我们来分享一些使用消息分块的最佳实践
<a name="PVXSJ"></a>
### 不要在消息中设置过大的元数据
<a name="rPjrd"></a>
### 在主题级别的maxMessageSize限制
<a name="ooB8J"></a>
### 消费者的最佳实践
<a name="psOpM"></a>
### debugging的最佳实践
<a name="D4HXT"></a>
### 
<a name="zRSAe"></a>
### 
<a name="U1Xl3"></a>
## 
