# Beam SDK 核心包(/sdks/java/core/)走读

作者：郭亚峰（默岭）

这个笔记会记录对SDK核心包的一个大致走读，对应的Java包名为： org.apache.beam.sdk。内容部分来自于对JavaDoc的阅读和翻译，部分是阅读总结。

## 索引



[TOC]

## 概览

| 子包名         | 内容         | 说明                                       |
| ----------- | ---------- | ---------------------------------------- |
| annotations | SDK里使用到的注解 | 目前只有一个注解Experimental。这个注解用来说明该API将来可能会有不兼容的变动，甚至这个API本身可能会被取消。一般而言，对应用来说，使用Experimental的接口是可以接受的，不过后期一旦接口有变动，应用本身也需要进行修改。一般不建议“库”组件使用Experimental API，因为这个可能导致改变接口的代价很大，且影响范围容易不可控。 |
| coders      | 数据序列化，反序列化 | 当pipeline从源头读取数据，或者把数据写入落地时，或者数据在pipeline中夸机器进行传递时，数据都是以二进制流的方式传输。而coder负责把Java对象转换成二进制流，或者从二进制流转换成Java对象。 这个包里包含了常见Java对象的Coder，也包含Avro等Coder可以处理大多数Java对象。 |
| io          | 数据输入与输出    | io主要用来把pipeline和外部数据系统进行集成。其中包括Read, Write这两个Transform，来分别从源头读数据，以及把数据写入落地存储中。而其他的Source, Sink抽象分别代表了数据的源头，处理结果的存储等。该包也包含一些简单的外部系统集成实现，如文件系统，如Google Cloud的消息队列集成等。 |
| metrics     | 监控指标       | 这个包用来暴露piple的执行统计信息，用来做监控，了解作业的执行情况。Metrics的信息也可以从pipeline的执行结果中查询到。（PipelineResult) |
| options     | 作业运行参数     | options用来定义pipleline的运行配置参数。它封装了用来描述pipeline如何运行的参数。有PipelineOptionsFactory创建。 |
| runners     | Runner相关   | 定义各种runner(??)。在Option中可以指定runner        |
| testing     | 单元测试       | 一方面定义了单元测试用的工具类，另一方面也提供了SDK, PTransform的使用例子。 |
| transforms  | 数据处理转换     | 可能是对应用开发者最重要的包。定义了一系列的PTransform，用来做数据转换处理。一个Transform一般有两个参数，第一个参数代表传入的数据，第二个参数代表传出的数据。而PTransform就是把输入数据转换为输出数据。常见的PTransform包括  TextIO.Read, ParDo, GroupByKey, join.CoGroupByKey, Combine, Count，TextIO.Write等。可以把现有的简单PTransform组合成复杂的PTransform， 也可以开发全新的PTransform来实现具体应用的逻辑。 |
| util        | Runner的工具类 | 定义Runner使用到的工具类。                         |
| values      | 定义数据结构     | 定义PCollection和其他Pipeline构建中使用到的数据结构。主要包括：PCollection，不可变的类型为T的集合。PCollectionView，不可变的PCollection的视图，用来作为ParDo的补充输入(Side Input)。PCollectionTuple，混杂了多个PCollection的数据元组。当PTransform需要多个Pcollection作为输入或输出时使用PCollectionTuple。 PCollectionList，一个混杂了多个PCollection的List。例如，Flatten就使用PCollectionList作为输入。  KV和TimestampedValue是其他两种常用的数据结构，分别在按键分组流和窗口、乱序处理流执行中使用。 |
| 主包类         |            | 主包下面也包含了一些类，比如Pipeline, PipelineResult代表了数据处理作业和处理结果的抽象。也包含了AggreagorValues相关的一些抽象（无法理解为什么这几个类要放到主包里，也许应该移到Aggegator的定义所在的transforms包当中？ |



##values包

我们先来展开了解values包，对Beam中的数据结构做一个探讨。
![values](D:\code\Apache-beam\incubator-beam-master\sdks\java\core\src\main\java\org\apache\beam\sdk\values\values.png)

这是values包的类图。P是Pipeline的缩写。简单来说，所有P打头类的都在以PInput和POut为根的继承树中。

***

**PInput (接口)**：

所有可能成为PTransform输入的类都必须实现的接口。包含三个基本的方法

* getPipeline(); 返回PInut所属的Pipeline
* expand(? extemds PValue): 如果当前的PInput是组合过的数据结构（如PCollectionTuple，PCollectionList），那么把它展开为最基本的PValue。如果当前的PInput已经是PValue，那么还是返回PValue。
* finishSpecifying: 结束PInput的构建，让它能够被PTransfrom使用。PTransform的apply()方法会自动调用这个方法，因此不需要应用手动调这个函数。

**POutput (接口)**:

所有可能成为PTransform输出的类都必须实现的接口。包含

* getPipeline(): 同上
* expand (? extends PValue) 同上。
* recordAsOutput(AppliedPTransform<?, ?, ?> transform) 用来记录该POutput是否是PTransform输出的一部分。如果是组合的POutput（如PCollectionTuple，PCollectionList），那么应该对每一个子元素调用这个方法。这个方法是框架内部方法，应用不需显式调用。
* finishSpecifyingOutput： 同上。完成数据的处理，让数据能成为下一步的输入或者成为输出结果。另外也用来确保Coder已经被指定，或者有默认的Coder。该方法在当前输出被用作下一步的输入，或者run方法调用时触发，不需要显式调用。

***

**PValue (接口)**  *继承了Pinput和POut接口*

PValue同时继承了PInput 和POut，和这两个接口一起定义了所有在Pipeline中传输的数据的共性。这个接口增加了下述两个方法的申明。

* getName(): 获得当前组件的名字。
* AppliedPTransform<?, ?, ?> getProducingTransformInternal()： 获得以该PValue为输出的AppliedPTransform。内部使用为主。

**PCollectionList** *实现了PInput和POut接口*

PCollectionList是一个不可变列表，是由PCollection组成的一个列表，但每个PCollection里的元素类型T可以不同。举例来说，Flatten（一种PTransform）使用了PColletionList作为输入，而Partition（一种PTransform）使用它作为输出。
代码例子：

```java
PCollection<String> pc1 = ...;
PCollection<String> pc2 = ...;
PCollection<String> pc3 = ...;

// Create a PCollectionList with three PCollections:
PCollectionList<String> pcs = PCollectionList.of(pc1).and(pc2).and(pc3);

// Create an empty PCollectionList:
Pipeline p = ...;
PCollectionList<String> pcs2 = PCollectionList.<String>empty(p);

// Get PCollections out of a PCollectionList, by index (origin 0):
PCollection<String> pcX = pcs.get(1);
PCollection<String> pcY = pcs.get(0);
PCollection<String> pcZ = pcs.get(2);

// Get a list of all PCollections in a PCollectionList:
List<PCollection<String>> allPcs = pcs.getAll();
```
PCollectionList 首先包含了多个从PCollection构造 PCollectionList的方法

* empty：构造一个空的PCollectinList
* of(PCollection)：返回一个新的PCollectionList 的Singlton，只包含一个传入的PCollection

如果要构造包含多个PCollection的，那么可以用

* of(Iterable <PCollection <T>>)：需要注意的是Iterable 里包含的所有PCollection应该属于同一个Pipeline。否则会抛异常。

用and可以比较灵活，对一个已经存在的PCollectionList一次增加一个元素或多个元素

* and(PCollection) 和 and(Iterable <PCollection <T>>) 。同样要求这些元素必须是属于同一个pipeline。


如上面的例子所示，可以用get, getAll()来获得PCollectionList的内容，用size()可以知道PCollectionList包含了多少个PCollection。

而apply方法接受一个tranform，并把这个tranform应用到自身上。

其余的方法是对继承了的接口进行实现。实现的方法基本都是转调每个PCollection元素对应的实现，不再一一阐述。

**PCollectionTuple** *实现了PInput和POut接口*

和PCollectionList非常相似，包含的方法也比较接近。唯一不同的是PCollectionTuple在内部使用了LinkedHashMap，存放着以TupleTag<?>为键，以PCollection为值的内部数据。

---

