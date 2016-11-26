# Beam SDK 核心包简单走读

包名： org.apache.beam.sdk

## 概览

| 子包名         | 内容         | 说明                                       |
| ----------- | ---------- | ---------------------------------------- |
| annotations | SDK里使用到的注解 | 目前只有一个注解Experimental。这个注解用来说明该API将来可能会有不兼容的变动，甚至这个API本身可能会被取消。一般而言，对应用来说，使用Experimental的接口是可以接受的，不过后期一旦接口有变动，应用本身也需要进行修改。一般不建议“库”组件使用Experimental API，因为这个可能导致改变接口的代价很大，且影响范围容易不可控。 |
| coders      | 数据序列化，反序列化 | 当pipeline从源头读取数据，或者把数据写入落地时，或者数据在pipeline中夸机器进行传递时，数据都是以二进制流的方式传输。而coder负责把Java对象转换成二进制流，或者从二进制流转换成Java对象。 这个包里包含了常见Java对象的Coder，也包含Avro等Coder可以处理大多数Java对象。 |
| io          | 数据输入与输出    | io主要用来把pipeline和外部数据系统进行集成。其中包括Read, Write这两个Transform，来分别从源头读数据，以及把数据写入落地存储中。而其他的Source, Sink抽象分别代表了数据的源头，处理结果的存储等。该包也包含一些简单的外部系统集成实现，如文件系统，如Google Cloud的消息队列集成等。 |
| metrics     |            |                                          |
| options     |            |                                          |
| runners     |            |                                          |
| testing     |            |                                          |
| transforms  |            |                                          |
| util        |            |                                          |
| values      |            |                                          |
| 主包类         |            |                                          |



