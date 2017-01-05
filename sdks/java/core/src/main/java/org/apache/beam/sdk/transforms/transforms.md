# Beam SDK 之 transforms

对Beam的SDK中的Transform包（可能是所有使用Beam开发应用程序的最常用的包吧）的代码随意走读，JavaDoc走读笔记。



# 普通转换计算 

| 类名                     | 作用                                       |
| ---------------------- | ---------------------------------------- |
| Filter                 | 过滤记录。包含多种过滤方法，如（by, greaterThan, greaterThanEq..) |
| FlatMapElements        | 一条输入记录可能产生0到多条输出记录                       |
| MapElements            | 一条输入记录只能产生1条输出记录                         |
| ParDo                  | 超级通用的方法。里面包含setup, teardown, startbundle等方法，可以支持比较复杂的生命周期操作。 |
| PTransform             | 所有Transform的父类                           |
| Regex                  | 同Filter，不过是使用正则表达式进行过滤。 支持matches, matchesKV, find等等。 |
| SimpleFunction         | SerializableFunction的默认实现，从而可以支持Lambda表达式操作 |
| SerializableFunction   | 一个包含apply方法的可序列化的接口                      |
| SerializableComparator | 接口，用来做值的比较用                              |

# 流数据转换

| 类名             | 作用                                       |
| -------------- | ---------------------------------------- |
| Create         | 可以按照指定数据生成PCollection，在测试，Mock数据源的时候经常用到 |
| Flatten        | 支持Union all操作的关键。可以把PCollectionList转换为PCollection |
| WithKeys       | 转换数据流从不带key的数据流转为带key的数据流                |
| Values         | 提取Value形成新的数据流                           |
| Keys           | 提取Key形成新的数据流                             |
| KvSwap         | 把KV的 key和value互换                         |
| Partition      | 把数据流切分成不同的partition                      |
| WithTimestamps | 为数据流附加上时间戳                               |
| View           | 把流转为静态的视图。详细可以参考PCollectionViews         |
| Sample         | 对数据流进行取样，获得更小的数据量。                       |

# 汇总计算

从类列表上可以看到聚合相关的类有4中类别。一种是Combine，一种是CombineFns，一种是CombineWithContext。最后是各种已经实现的汇总统计方法，如Sum, Min, Max,UV估算，采样等等。我们先来过一下內建的统计方法。然后再看看通用性强的Combine, CombineFns，

## 内置聚合

| 函数所在类名               | 描述                                       |
| -------------------- | :--------------------------------------- |
| Count                | 内置了CountFn，用来做对任意传入元素的计数。默认不排重。同时提供了Count..globally(), Count.perKey, Count.perElement来分别做全局计数，按key计数，和按值出现次数计数。 |
| Distinct             | 严格来说不是聚合函数，但实际中计算DAU类指标肯定需要的。Distinct的去重依据默认按Coder之后的二进制结果比较。也可以实现WithRepresentativeValues方法来给出一个新的值来参与排重。Distinct默认需要窗口，排重限定在一个窗口内不重复。输出流会带上窗口的结束时间作为时间戳，带上输入流的WinFn作为窗口分配函数。 |
| Latest               | 内置LatestFn，支持Latest.globally, Latest.perKey |
| Max                  | Max支持的功能也比较全面，有根据不同数据类型的比较，也可以根据自定义函数的返回值进行比较。另外它的naturalOrder的实现依赖于Top.Largest，本质上相当于采用了Java对象的compareTo进行比较。 |
| Mean                 | 取均值，同样支持globally 和perkey                 |
| Min                  | 和max相同                                   |
| Sum                  | 整体设计和max比较类似，也提供了基于BinaryCombineFn的聚合SumIntegerFn，SumLongFn， SumDoubleFn等。 |
| Top                  | 求取最大或最小值。要就结果列表必须是能够cache到内存中的。有Top.largest, Top.smallest, Top.largestPerKey, Top.smallestPerkey. |
| ApproximateQuantiles | Ntiles 数据分布估计。                           |
| ApproximateUnique    | 采用基数估计法，偏差估计会比较大。实际用到的话可以观察下偏差情况。同样支持perKey, globally |

## Combine

Combine因为构造器是私有的，无法初始化。所以本身只是一个容器，有意义的是它内部定义的一系列的静态类。这也是Beam中常见的设计模式。把类当做命名空间来用，而这个命名空间里的不同语义实现作为静态的内部类进行定义。这样SDK的使用者能够获得相对更能体现语义含义的接口。不好的地方就是每个类的代码都相当长，对SDK的开发者而言有一定维护复杂度的提升。

### Combine中的 CombineFn

#### Combine.CombineFn

 CombineFn<InputT, AccumT, OutputT>} 定义了如何把一组类型为InputT的集合汇总为一条类型为OutputT的输出。在汇总的过程中可能还要需要用到一到多个类型为AccumT的中间可变状态累加器(Accumulator)。整个汇总的过程大概可以分为以下几个步骤。

1. 输入值(类型为InputT) 分到一个或多个批次中。对每一个批次，调用createAccumulator来创建一个全新的可变的累加器。累加器初始化为代表0个输入值合并后的结果。
2. 对每个输入，系统调用addInput来把输入值累加到本批次的累加器上去。累加器可能仅仅是把新的值保存在列表中（即：AccumT == List<InputT>），也可能是做一些运算得出一个中间的结果值。
3. mergeAccumulators方法用来把不同批次的累加器合并起来成为一个唯一的输出累加器。这个操作可能会被反复调用，直到获得唯一的累加器。
4. 最后extractOutput方法从第三个步骤中获得的累加器中提取出数据，并转换为OutputT类型输出。

```Java
 public class AverageFn extends CombineFn<Integer, AverageFn.Accum, Double> {
   public static class Accum {
     int sum = 0;
     int count = 0;
   }
   public Accum createAccumulator() {
     return new Accum();
   }
   public Accum addInput(Accum accum, Integer input) {
       accum.sum += input;
       accum.count++;
       return accum;
   }
   public Accum mergeAccumulators(Iterable<Accum> accums) {
     Accum merged = createAccumulator();
     for (Accum accum : accums) {
       merged.sum += accum.sum;
       merged.count += accum.count;
     }
     return merged;
   }
   public Double extractOutput(Accum accum) {
     return ((double) accum.sum) / accum.count;
   }
 }
 PCollection<Integer> pc = ...;
 PCollection<Double> average = pc.apply(Combine.globally(new AverageFn()));
 }
```

Combine.Globally, Combine.PerKey和Combine.GroupedValues都可以使用上述的Combine Function。从它们派生的PTransform只能支持符合交换律（Commutative)和结合律（associative)的运算逻辑。需要满足结合律是因为输入值先是被分到了很多小组，然后计算出中间结果进行合并，类似一个任意的数状结构。必须满足交换律是因为我们在把输入值分组时不考虑输入值的顺序性。

#### Combine.KeyedCombineFn

KeyedCombineFn和CombineFn基本一致。唯一的区别在于输入值是key value对。汇总是按key进行的，因此key伴随了所有主要的接口和动作。

下面的例子把所有key相同的字符串拼接到一个长字符串中然后输出。


```JAVA
    public class ConcatFn
        extends KeyedCombineFn<String, Integer, ConcatFn.Accum, String> {
      public static class Accum {
        String s = "";
      }
      public Accum createAccumulator(String key) {
        return new Accum();
      }
      public Accum addInput(String key, Accum accum, Integer input) {
          accum.s += "+" + input;
          return accum;
      }
      public Accum mergeAccumulators(String key, Iterable<Accum> accums) {
        Accum merged = new Accum();
        for (Accum accum : accums) {
          merged.s += accum.s;
        }
        return merged;
      }
      public String extractOutput(String key, Accum accum) {
        return key + accum.s;
      }
    }
    PCollection<KV<String, Integer>> pc = ...;
    PCollection<KV<String, String>> pc2 = pc.apply(
        Combine.perKey(new ConcatFn()));
    } 
```



#### Combine.IterableCombineFn

IterableCombineFn能够把一个普通的接受Iterable<v>的 Serilizable Function包装成一个简单接受V的CombineFn。 是一个内部经常使用的工具类。

SimpleCombineFn是IterableCombineFn的早期版本。目前已经声明为废弃中的接口。不建议再使用SimpleCombineFn。

#### Combine.AccumulatingCombineFn

这个类预定义了累加器的接口（AccumulatingCombineFn.Accumulator)，并且把相关的处理逻辑进行了封装，因此使用上比直接使用CombineFn相对要简明一点。比如说，上面的例子用AccumulatingCombineFn可以实现的稍微更简短一点：

```Java
public class AverageFn
  extends AccumulatingCombineFn<Integer, AverageFn.Accum, Double> {
public Accum createAccumulator() {
  return new Accum();
}
public class Accum
    extends AccumulatingCombineFn<Integer, AverageFn.Accum, Double>
            .Accumulator {
  private int sum = 0;
  private int count = 0;
  public void addInput(Integer input) {
    sum += input;
    count++;
  }
  public void mergeAccumulator(Accum other) {
    sum += other.sum;
    count += other.count;
  }
  public Double extractOutput() {
    return ((double) sum) / count;
  }
}
}
PCollection<Integer> pc = ...;
PCollection<Double> average = pc.apply(Combine.globally(new AverageFn()));
```



#### Combine.BinaryCombineFn 和BinaryCombineIntegerFn, BinaryCombineDoubleFn, BinaryCombineLongFn

这几个CombineFn都是为了便利于定义支持两两合并操作的汇总计算。BinaryCombineFn是一个抽象类，而其余几个是针对常见基本类型的抽象类。注意它们之间不存在继承关系。

Holder是BinaryCombineFn使用的累加器的类型，用来存储中间状态用。

而几个CombineFn中都有identity()方法，这个可能比较难以理解，这里拿出来单独说一下。这个方法用来返回一个初始值，用来和第一条到达的输入数据开始进行两两合并汇总。因此，针对你要实现的汇总计算，必须要谨慎地选择identity().

举个例子，如果你要实现加法，那么理想的identity()是返回0，而如果要实现乘法运算，那么合适的identity()是1。就像JavaDoc中注释的那样，一个identity()的返回值e是应该对所有的可能输入值x都满足：

```Java
apply(e, x) == apply(x, e) == x
```

即：满足交换律，满足参与运算后不会影响运算汇总结果。

其余的接口，方法和CombineFn相同，不再繁叙。

### Combine中的PTransform

#### Combine.Globally和Combine.GloballyAsSingletonView

和上面的CombineFn不同，这两者都是PTransform，而不单单是CombineFn。 PTransform使用CombineFn，按照CombineFn制定的逻辑处理数据并把结果返回。这两者大概是这样一个关系。

Globally对每个窗口中的数据进行全局汇总（无维度参加），产生一条输出数据。输出数据的类型（OutputT）可能和输入数据的类型相同，也可以是完全不同。这个汇总的逻辑由构造函数中传入的CombineFn进行处理。常见的聚合操作有求和，求最大最小值，均值等等。

例子

```Java
PCollection<Integer> pc = ...;
   PCollection<Integer> sum = pc.apply(
   		Combine.globally(new Sum.SumIntegerFn()));
```

合并的操作可以并行执行。首先是每一部分输入分别计算汇总得到中间结果。然后中间结果进一步进行合并汇总。整个合并过程如同一颗树一样，从叶子节点开始慢慢合并，直到得到一个唯一的结果。

如果计算采用的窗口是全局窗口（GlobalWindos)， 那么当数据输入为空时，GlobalWindow的一个默认值会成为默认数据输出。而如果这个结果需要流入其他类型的窗口进行进一步处理，那么你应该调用withoutDefaults（告诉系统如果没有输入那么就不要吐出默认输出）或者是asSingletonView（返回GloballyAsSingletonView）。这是因为默认值无法自动赋给一个单独非全局性窗口。

默认地，输出的Coder和CombineFn的输出的Coder一致。

后面还可以参考PerKey和 GroupedValues，它们对处理K,V类型的数据非常有用。

GloballyAsSingletonView和 Globally完全一致。区别在于前者返回的是PCollectionView，而后者是PCollection。

这里单独提一下fanout。fanout机制是为了降低全局汇总节点的压力，在汇总前增加一些中间节点进行并行汇总，然后把结果输出给最后的全局节点进行最后汇总。fanout参数设置了中间节点的个数。

#### Combine.PerKey Combine.PerKeyWithHotKeyFanout

Combine.PerKey接受KV形式的输入，按Key对数据进行分组，按指定的Combine函数对数据进行聚合，返回KV形式的汇总结果。输入输出数据的K相同，V的数据类型一般也相同。

Combine.PerKey可以看做是GroupByKey + Combine.GroupedValues的快捷形式，关于如何进行Key的等值比较和默认的输出Coder如何确定可以参考这两个组件的定义。

```Java
 PCollection<KV<String, Double>> salesRecords = ...;
 PCollection<KV<String, Double>> totalSalesPerPerson =
     salesRecords.apply(Combine.<String, Double, Double>perKey(
         new Sum.SumDoubleFn()));
 }
```
每一个输出的元素都带有和输入流一样的窗口，时间戳则是窗口结束边沿的时间戳。并且PCollection上也有和输入相同的时间窗口函数。如果下游有新的汇总处理，这些窗口属性会影响新的汇总。

而PerKeyWithHotKeyFanout能够自动对热键进行Fanout操作，避免数据倾斜带来的影响。具体代码比较长，细节这里就不一一覆盖了（后续仔细阅读后可以再补充这一部分。而且前面Globally的fanout的支持，也是在这一部分当中实现的）。

#### Combine.GroupedValues

GroupedValues是针对已经按Key已经分好组的数据按指定的CombineFn进行汇总操作的PTransform。因此，它只接受PCollection<KV<K,Iterable<InputT>>>类型的输入。一般和GroupByKey一起工作，而接受的CombineFn是KeyedCombineFn。输出通常也是带Key 的PCollection，也就是PCollection<KV<K,OutputT>>。通常InputT和OutputT相同，但不是必须的。

例子： 

```Java
 PCollection<KV<String, Integer>> pc = ...;
 PCollection<KV<String, Iterable<Integer>>> groupedByKey = pc.apply(
     new GroupByKey<String, Integer>());
 PCollection<KV<String, Integer>> sumByKey = groupedByKey.apply(
     Combine.<String, Integer>groupedValues(
         new Sum.SumIntegerFn()));
 } 
```
上面曾经说过，PerKey是GroupedByKey和Combine.GroupedValues的合体。
整个汇总的过程中，每个Key对应的汇总是独立进行的，而同一个Key的汇总也是可以并行进行的，采用的方式就是前面提到过的树状汇总方法。
默认的输出的Coder和输入的Coder方式是一样的，从输入推断而来。
每一个输出的元素都带有和输入流一样的窗口，时间戳则是窗口结束边沿的时间戳。并且PCollection上也有和输入相同的时间窗口函数。如果下游有新的汇总处理，这些窗口属性会影响新的汇总。

## CombineFnBase

CombineFnBase为CombineFn 提供了一些共享的接口和虚类。下面简单过一下。它是CombineFns的基础。Combine中也有使用到。一般来说，应用开发者没必要直接实现或者扩展它们，应该直接使用SDK中已经有的实现。

### GlobalCombineFn 和 AbstractGlobalCombineFn

GloballyCombineFn<InputT, AccumT, OutputT>} 定义了如何把一组类型为InputT的集合汇总为一条类型为OutputT的输出。在汇总的过程中可能还要需要用到一到多个类型为AccumT的中间可变状态累加器(Accumulator)。

接口GlobalCombineFn 里有两个重要的方法定义。一个是getAccumulatorCoder， 一个是getDefaultOutputCoder。都是和Coder相关的。其中accumulator累加器的Coder尤为关键。因为Accumulator相当于分布式计算中Shuffle的步骤，涉及大量的数据传输，因此高效的Coder对于作业整体的性能非常关键。

AbstractGlobalCombineFn 为getAccumulatorCoder提供了默认实现，使用InputT的Coder来进行推断获得。推断的方式是从所属pipeline的CodeRegistery中按InputT的类型去进行查找。

getDefaultOutputCoder是获得OutputT的Coder。同样 AbstractGlobalCombineFn提供了默认的推断实现。推断的方式是从所属pipeline的CodeRegistery中按InputT的Coder类型和AccumT的Coder类型去进行查找。

### PerKeyCombineFn 和AbstractPerKeyCombineFn

和两者和上面的区别在于有了Key的类型。其他均无太大区别。其实GlobalCombineFn和PerKeyCombineFn本身也可以互相转换。如果我们人为指定一个具体的key，就可以把PerKeyCombineFn转换为GlobalCombineFn。如果KeyedFn里面所有处理都忽略掉key，那么GlobalCombineFn就可以转换为KeyedFn。

## CombineFnWithContext

这个内包含两个主要的内部类，CombineFnWithContext 和 KeyedCombineFnWithContext。这两个类分别是AbstractGlobalCombineFn和AbstractPerKeyCombineFn的子类。 区别在于每个主要的方法上都附带了Context， 可以通过它获得PipelineOptions和sideInput。

另外它定义了一个标机接口RequiresContextInternal，CombineFnWithContext 和KeyedCombineFnWithContext都实现了这个标记接口。

## CombineFns

这个类是非常关键的一个工具类，用来创建CombineFn的实例。它同样包含很多内部类和一些重要的创建组合CombineFn的方法，如compose(), composeKeyed() 等。这个工具类里定义的CombineFn主要有四种。ComposedCombineFn，ComposedCombineFnWithContext，ComposedKeyedCombineFn，ComposedKeyedCombineFnWithContext。这几种CombineFn的区别可以从名字上就看出来。 有key，还是global, 有context还是没有。除此之外四者间基本没有区别。下面重点看一下ComposedCombineFn。

ComposedCombineFn是多个CombineFn组合在一起之后形成的一个复合CombineFn。想象一下现实中的数据处理场景，对同一批的数据可能会有多个汇总加工的需求，比如说要count某个字段，同时要求和等等。那么这个时候就需要把多个CombineFn组合成一个ComposedCombineFn。

ComposedCombineFn 三个比较重要的成员变量，其中extractInputFns是用来对输入数据DataT进行转换用的。因为ComposedCombineFn 包含了多个CombineFn， 每个CombineFn需要的输入类型可能不尽相同。因此对于每个CombineFn可以定义一个SerializableFunction进行数据类型的适配动作。combineFns 则是构成ComposedCombineFn 的各个成员CombineFn。每一个CombineFn都对DataT进行加工，产生一个输出。而每一个输出都有一个TupleTag与之进行关联。outputTags就是所有输出的TupleTag组成的列表。

ComposedCombineFn的with方法会首先检查传入的TupleTag是否已经存在。没有的话它就把新传入的extractInput, CombineFn, OutputTag追加到原有的列表中，并返回一个新的ComposedCombineFn。 整体上是一个Immutable的设计方式。

ComposeCombineFnBuilder可以根据传入参数的不同分别调用ComposedCombineFn或ComposedCombineFnWithContext的with方法来常见对应的ComposedCombineFn或者ComposedCombineFnWithContext。

而ComposedKeyedCombineFn，ComposedKeyedCombineFnWithContext，ComposeKeyedCombineFnBuilder主要是增加了K key，其他并无本质区别。不在赘述。

此外还有CoCombineResult内部类。用来对输出的valuesMap进行加工，补齐结果为NULL的数据。另外提供了按TupleTag获取具体某个单个结果项的能力。而ProjectionIterable提供了对某个具体输入字段进行Iterable的功能。

# 窗口

# Join



待续









