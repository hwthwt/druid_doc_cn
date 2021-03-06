# 架构更改

数据源的模式可以随时更改，而Druid支持不同的模式。

## 替换细分

Druid使用数据源，间隔，版本和分区号唯一标识段。如果在某个粒度的时间内创建了多个段，则分区号仅在段ID中可见。例如，如果您有每小时段，但是您在一小时内拥有的数据多于单个段可以容纳的数据，则可以在同一小时创建多个段。这些段将共享相同的数据源，间隔和版本，但具有线性增加的分区号。

```text
foo_2015-01-01/2015-01-02_v1_0
foo_2015-01-01/2015-01-02_v1_1
foo_2015-01-01/2015-01-02_v1_2
```

在上面的示例段中，dataSource = foo，interval = 2015-01-01 / 2015-01-02，version = v1，partitionNum = 0.如果在稍后的某个时间点，您使用新架构重新索引数据，新创建的段将具有更高的版本ID。

```text
foo_2015-01-01/2015-01-02_v2_0
foo_2015-01-01/2015-01-02_v2_1
foo_2015-01-01/2015-01-02_v2_2
```

德鲁伊批量索引（基于Hadoop或基于IndexTask）可以逐个间隔地保证原子更新。在我们的示例中，在将所有`v2`段`2015-01-01/2015-01-02`加载到Druid集群中之前，查询将专门使用`v1`段。`v2`加载并查询所有段后，所有查询都会忽略`v1`段并切换到`v2`段。不久之后，这些`v1`段将从群集中卸载。

请注意，跨越多个段间隔的更新在每个间隔内仅是原子的。它们在整个更新过程中不是原子的。例如，您有以下段：

```text
foo_2015-01-01/2015-01-02_v1_0
foo_2015-01-02/2015-01-03_v1_1
foo_2015-01-03/2015-01-04_v1_2
```

`v2`段将在构建后立即加载到群集中，并`v1`在段重叠的时间段内替换段。在v2段完全加载之前，您的群集可能包含`v1`和`v2`段的混合。

```text
foo_2015-01-01/2015-01-02_v1_0
foo_2015-01-02/2015-01-03_v2_1
foo_2015-01-03/2015-01-04_v1_2
```

在这种情况下，查询可能会混合使用`v1`和`v2`分段。

## 细分中的不同模式

同一数据源的德鲁伊段可能有不同的模式。如果字符串列（维度）存在于一个段中而不存在于另一个段中，则涉及两个段的查询仍然有效。缺少维度的段的查询将表现为维度仅具有空值。类似地，如果一个段具有数字列（度量）而另一个段没有，则对缺少度量的段的查询通常将“做正确的事”。对此缺失度量标准的聚合表现为缺少度量标准。