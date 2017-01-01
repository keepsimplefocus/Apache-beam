# Beam的Transform

对Beam的SDK中的Transform包（可能是所有使用Beam开发应用程序的最常用的包吧）的代码随意走读，JavaDoc走读笔记。

## Class概览

### 类图 

（暂时空缺）

### 类列表

| 类名                                       |
| ---------------------------------------- |
| AggregatorRetriever                      |
| AppliedPTransform                        |
| ApproximateQuantiles                     |
| ApproximateQuantiles.ApproximateQuantilesCombineFn |
| ApproximateUnique                        |
| ApproximateUnique.ApproximateUniqueCombineFn |
| ApproximateUnique.ApproximateUniqueCombineFn.LargestUnique |
| Combine                                  |
| Combine.AccumulatingCombineFn            |
| Combine.BinaryCombineDoubleFn            |
| Combine.BinaryCombineFn                  |
| Combine.BinaryCombineIntegerFn           |
| Combine.BinaryCombineLongFn              |
| Combine.CombineFn                        |
| Combine.Globally                         |
| Combine.GloballyAsSingletonView          |
| Combine.GroupedValues                    |
| Combine.Holder                           |
| Combine.IterableCombineFn                |
| Combine.KeyedCombineFn                   |
| Combine.PerKey                           |
| Combine.PerKeyWithHotKeyFanout           |
| Combine.SimpleCombineFn                  |
| CombineFnBase                            |
| CombineFns                               |
| CombineFns.CoCombineResult               |
| CombineFns.ComposeCombineFnBuilder       |
| CombineFns.ComposedCombineFn             |
| CombineFns.ComposedCombineFnWithContext  |
| CombineFns.ComposedKeyedCombineFn        |
| CombineFns.ComposedKeyedCombineFnWithContext |
| CombineFns.ComposeKeyedCombineFnBuilder  |
| CombineWithContext                       |
| CombineWithContext.CombineFnWithContext  |
| CombineWithContext.Context               |
| CombineWithContext.KeyedCombineFnWithContext |
| Count                                    |
| Count.PerElement                         |
| Create                                   |
| Create.TimestampedValues                 |
| Create.Values                            |
| DoFn                                     |
| DoFn.FakeExtraContextFactory             |
| DoFn.ProcessContinuation                 |
| DoFnAdapters                             |
| DoFnTester                               |
| Filter                                   |
| FlatMapElements                          |
| FlatMapElements.MissingOutputTypeDescriptor |
| Flatten                                  |
| Flatten.FlattenIterables                 |
| Flatten.FlattenPCollectionList           |
| GroupByKey                               |
| Keys                                     |
| KvSwap                                   |
| Latest                                   |
| Latest.LatestFn                          |
| MapElements                              |
| MapElements.MissingOutputTypeDescriptor  |
| Max                                      |
| Max.MaxDoubleFn                          |
| Max.MaxFn                                |
| Max.MaxIntegerFn                         |
| Max.MaxLongFn                            |
| Mean                                     |
| Min                                      |
| Min.MinDoubleFn                          |
| Min.MinFn                                |
| Min.MinIntegerFn                         |
| Min.MinLongFn                            |
| OldDoFn                                  |
| ParDo                                    |
| ParDo.Bound                              |
| ParDo.BoundMulti                         |
| ParDo.Unbound                            |
| ParDo.UnboundMulti                       |
| Partition                                |
| PTransform                               |
| RemoveDuplicates                         |
| RemoveDuplicates.WithRepresentativeValues |
| Sample                                   |
| Sample.FixedSizedSampleFn                |
| Sample.SampleAny                         |
| SimpleFunction                           |
| Sum                                      |
| Sum.SumDoubleFn                          |
| Sum.SumIntegerFn                         |
| Sum.SumLongFn                            |
| Top                                      |
| Top.Largest                              |
| Top.Smallest                             |
| Top.TopCombineFn                         |
| Values                                   |
| View                                     |
| View.AsIterable                          |
| View.AsList                              |
| View.AsMap                               |
| View.AsMultimap                          |
| View.AsSingleton                         |
| View.CreatePCollectionView               |
| ViewFn                                   |
| WithKeys                                 |
| WithTimestamps                           |





# 汇总计算

从类列表上可以看到Combine相关的类有4中类别。一种是Combine，一种是CombineFns，一种是CombineWithContext。最后是各种已经实现的汇总统计方法，如Sum, Min, Max,UV估算，采样等等。我们先来看看通用性强的Combine, CombineFns，然后再过一下內建的统计方法。

## Combine

Combine因为构造器是私有的，无法初始化。所以本身只是一个容器，有意义的是它内部定义的一系列的静态类。这也是Beam中常见的设计模式。把类当做命名空间来用，而这个命名空间里的不同语义实现作为静态的内部类进行定义。这样SDK的使用者能够获得相对更能体现语义含义的接口。不好的地方就是每个类的代码都相当长，对SDK的开发者而言有一定维护复杂度的提升。

### Combine.CombineFn

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

### Combine.KeyedCombineFn

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



### Combine.IterableCombineFn

IterableCombineFn能够把一个普通的接受Iterable<v>的 Serilizable Function包装成一个简单接受V的CombineFn。 是一个内部经常使用的工具类。

SimpleCombineFn是IterableCombineFn的早期版本。目前已经声明为废弃中的接口。不建议再使用SimpleCombineFn。

### Combine.AccumulatingCombineFn

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



### Combine.BinaryCombineFn 和BinaryCombineIntegerFn, BinaryCombineDoubleFn, BinaryCombineLongFn

这几个CombineFn都是为了便利于定义支持两两合并操作的汇总计算。BinaryCombineFn是一个抽象类，而其余几个是针对常见基本类型的抽象类。注意它们之间不存在继承关系。

Holder是BinaryCombineFn使用的累加器的类型，用来存储中间状态用。

而几个CombineFn中都有identity()方法，这个可能比较难以理解，这里拿出来单独说一下。这个方法用来返回一个初始值，用来和第一条到达的输入数据开始进行两两合并汇总。因此，针对你要实现的汇总计算，必须要谨慎地选择identity().

举个例子，如果你要实现加法，那么理想的identity()是返回0，而如果要实现乘法运算，那么合适的identity()是1。就像JavaDoc中注释的那样，一个identity()的返回值e是应该对所有的可能输入值x都满足：

```Java
apply(e, x) == apply(x, e) == x
```

即：满足交换律，满足参与运算后不会影响运算汇总结果。

其余的接口，方法和CombineFn相同，不再繁叙。

### Combine.Globally和Combine.GloballyAsSingletonView

和上面的CombineFn不同，这两者都是PTransform，而不单单是CombineFn。 PTransform使用CombineFn，按照CombineFn制定的逻辑处理数据并把结果返回。这两者大概是这样一个关系。

Globally对每个窗口中的数据进行全局汇总（无其他维度参加），产生一条输出数据。输出数据的类型（OutputT）可能和输入数据的类型相同，也可以是完全不同。这个汇总的逻辑由构造函数中传入的CombineFn进行处理。常见的聚合操作有求和，求最大最小值，均值等等。

例子

```Java
PCollection<Integer> pc = ...;
   PCollection<Integer> sum = pc.apply(
   		Combine.globally(new Sum.SumIntegerFn()));
```

合并的操作可以并行执行。首先是每一部分输入分别计算汇总得到中间结果。然后中间结果进一步进行合并汇总。整个合并过程如同一颗树一样，从叶子节点开始慢慢合并，直到得到一个唯一的结果。

如果输入窗口是全局窗口（GlobalWindos)， 那么当数据输入为空时，GlobalWindow的一个默认值会成为默认数据输出。而如果输入窗口是其他类型的窗口，那么你应该调用withoutDefaults或者是asSingletonView。这是因为默认值无法自动赋给一个单独的窗口。

默认地，输出的Coder和CombineFn的输出的Coder一致。

后面还可以参考PerKey和 GroupedValues，它们对处理K,V类型的数据非常有用。

