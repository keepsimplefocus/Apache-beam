# Transform Package JavaDoc

## Class概览

### 类图 

（暂时空缺）

### 类列表

| 类名                                       | 简单描述                                     | 备注    |
| ---------------------------------------- | ---------------------------------------- | ----- |
| AggregatorRetriever                      | 从DoFn中获取Aggregors。Aggegator是用来监控输入值得，可以从监控页面中看到Aggegator的当前值。 |       |
| AppliedPTransform                        | 代表了了把一个特定的PTransform应用到某一特定输入后产生的Transform。 | 一般内部用 |
| ApproximateQuantiles                     |                                          |       |
| ApproximateQuantiles.ApproximateQuantilesCombineFn |                                          |       |
| ApproximateUnique                        |                                          |       |
| ApproximateUnique.ApproximateUniqueCombineFn |                                          |       |
| ApproximateUnique.ApproximateUniqueCombineFn.LargestUnique |                                          |       |
| Combine                                  |                                          |       |
| Combine.AccumulatingCombineFn            |                                          |       |
| Combine.BinaryCombineDoubleFn            |                                          |       |
| Combine.BinaryCombineFn                  |                                          |       |
| Combine.BinaryCombineIntegerFn           |                                          |       |
| Combine.BinaryCombineLongFn              |                                          |       |
| Combine.CombineFn                        |                                          |       |
| Combine.Globally                         |                                          |       |
| Combine.GloballyAsSingletonView          |                                          |       |
| Combine.GroupedValues                    |                                          |       |
| Combine.Holder                           |                                          |       |
| Combine.IterableCombineFn                |                                          |       |
| Combine.KeyedCombineFn                   |                                          |       |
| Combine.PerKey                           |                                          |       |
| Combine.PerKeyWithHotKeyFanout           |                                          |       |
| Combine.SimpleCombineFn                  |                                          |       |
| CombineFnBase                            |                                          |       |
| CombineFns                               |                                          |       |
| CombineFns.CoCombineResult               |                                          |       |
| CombineFns.ComposeCombineFnBuilder       |                                          |       |
| CombineFns.ComposedCombineFn             |                                          |       |
| CombineFns.ComposedCombineFnWithContext  |                                          |       |
| CombineFns.ComposedKeyedCombineFn        |                                          |       |
| CombineFns.ComposedKeyedCombineFnWithContext |                                          |       |
| CombineFns.ComposeKeyedCombineFnBuilder  |                                          |       |
| CombineWithContext                       |                                          |       |
| CombineWithContext.CombineFnWithContext  |                                          |       |
| CombineWithContext.Context               |                                          |       |
| CombineWithContext.KeyedCombineFnWithContext |                                          |       |
| Count                                    |                                          |       |
| Count.PerElement                         |                                          |       |
| Create                                   |                                          |       |
| Create.TimestampedValues                 |                                          |       |
| Create.Values                            |                                          |       |
| DoFn                                     |                                          |       |
| DoFn.FakeExtraContextFactory             |                                          |       |
| DoFn.ProcessContinuation                 |                                          |       |
| DoFnAdapters                             |                                          |       |
| DoFnTester                               |                                          |       |
| Filter                                   |                                          |       |
| FlatMapElements                          |                                          |       |
| FlatMapElements.MissingOutputTypeDescriptor |                                          |       |
| Flatten                                  |                                          |       |
| Flatten.FlattenIterables                 |                                          |       |
| Flatten.FlattenPCollectionList           |                                          |       |
| GroupByKey                               |                                          |       |
| Keys                                     |                                          |       |
| KvSwap                                   |                                          |       |
| Latest                                   |                                          |       |
| Latest.LatestFn                          |                                          |       |
| MapElements                              |                                          |       |
| MapElements.MissingOutputTypeDescriptor  |                                          |       |
| Max                                      |                                          |       |
| Max.MaxDoubleFn                          |                                          |       |
| Max.MaxFn                                |                                          |       |
| Max.MaxIntegerFn                         |                                          |       |
| Max.MaxLongFn                            |                                          |       |
| Mean                                     |                                          |       |
| Min                                      |                                          |       |
| Min.MinDoubleFn                          |                                          |       |
| Min.MinFn                                |                                          |       |
| Min.MinIntegerFn                         |                                          |       |
| Min.MinLongFn                            |                                          |       |
| OldDoFn                                  |                                          |       |
| ParDo                                    |                                          |       |
| ParDo.Bound                              |                                          |       |
| ParDo.BoundMulti                         |                                          |       |
| ParDo.Unbound                            |                                          |       |
| ParDo.UnboundMulti                       |                                          |       |
| Partition                                |                                          |       |
| PTransform                               |                                          |       |
| RemoveDuplicates                         |                                          |       |
| RemoveDuplicates.WithRepresentativeValues |                                          |       |
| Sample                                   |                                          |       |
| Sample.FixedSizedSampleFn                |                                          |       |
| Sample.SampleAny                         |                                          |       |
| SimpleFunction                           |                                          |       |
| Sum                                      |                                          |       |
| Sum.SumDoubleFn                          |                                          |       |
| Sum.SumIntegerFn                         |                                          |       |
| Sum.SumLongFn                            |                                          |       |
| Top                                      |                                          |       |
| Top.Largest                              |                                          |       |
| Top.Smallest                             |                                          |       |
| Top.TopCombineFn                         |                                          |       |
| Values                                   |                                          |       |
| View                                     |                                          |       |
| View.AsIterable                          |                                          |       |
| View.AsList                              |                                          |       |
| View.AsMap                               |                                          |       |
| View.AsMultimap                          |                                          |       |
| View.AsSingleton                         |                                          |       |
| View.CreatePCollectionView               |                                          |       |
| ViewFn                                   |                                          |       |
| WithKeys                                 |                                          |       |
| WithTimestamps                           |                                          |       |