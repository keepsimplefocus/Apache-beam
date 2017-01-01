# Transform Package JavaDoc

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

    A {@code KeyedCombineFn<K, InputT, AccumT, OutputT>} specifies how to combine
    a collection of input values of type {@code InputT}, associated with
    a key of type {@code K}, into a single output value of type
    {@code OutputT}.  It does this via one or more intermediate mutable
    accumulator values of type {@code AccumT}.
       
    <p>The overall process to combine a collection of input
    {@code InputT} values associated with an input {@code K} key into a
    single output {@code OutputT} value is as follows:
       
    <ol>
       
    <li> The input {@code InputT} values are partitioned into one or more
    batches.
       
    <li> For each batch, the {@link #createAccumulator} operation is
    invoked to create a fresh mutable accumulator value of type
    {@code AccumT}, initialized to represent the combination of zero
    values.
       
    <li> For each input {@code InputT} value in a batch, the
    {@link #addInput} operation is invoked to add the value to that
    batch's accumulator {@code AccumT} value.  The accumulator may just
    record the new value (e.g., if {@code AccumT == List<InputT>}, or may do
    work to represent the combination more compactly.
       
    <li> The {@link #mergeAccumulators} operation is invoked to
    combine a collection of accumulator {@code AccumT} values into a
    single combined output accumulator {@code AccumT} value, once the
    merging accumulators have had all all the input values in their
    batches added to them.  This operation is invoked repeatedly,
    until there is only one accumulator value left.
       
    <li> The {@link #extractOutput} operation is invoked on the final
    accumulator {@code AccumT} value to get the output {@code OutputT} value.
       
    </ol>
       
    <p>All of these operations are passed the {@code K} key that the
    values being combined are associated with.
       
    <p>For example:
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

    <p>Keyed combining functions used by {@link Combine.PerKey},
    {@link Combine.GroupedValues}, and {@code PTransforms} derived
    from them should be <i>associative</i> and <i>commutative</i>.
    Associativity is required because input values are first broken
    up into subgroups before being combined, and their intermediate
    results further combined, in an arbitrary tree structure.
    Commutativity is required because any order of the input values
    is ignored when breaking up input values into groups.


### Combine.Globally

输入： PCollection<InputT>。 

输出： PCollection<OutputT>。 

输入输出的类型InputT和OutputT经常相同，不过这不是必须的。转换的逻辑由CombineFn(InputT, AccumT, OutputT)定义。常见的汇总统计函数有求和，求最大最小值，平均值，统计汇总，布尔操作等。如果输入数据使用的是GlobalWindows，那么如果输入为空，GlobalWindow会有一条默认的输出。如果你使用的是其他类型的窗口，那么必须调用withoutDefaults或者asSingletonView。因为默认值无法赋给一个单独的窗口。序列化用的Coder默认根据CombineFn的输出类型推断出来。

例子：PCollection<Integer> sum = pc.apply(Combine.globally(new Sum.SumIntegerFn()));