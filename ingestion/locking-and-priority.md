# 任务锁定和优先级

## 锁定

一旦霸王节点接受任务，该任务就会获取数据源的锁定以及任务中指定的时间间隔。

有两种锁类型，即*共享锁*和*独占锁*。

- 任务需要在读取间隔段之前获取共享锁。可以为相同的dataSource和interval获取多个共享锁。共享锁始终是可抢占的，但它们不会互相抢占。
- 任务需要在为一个时间间隔写入段之前获取独占锁。除了任务是发布段之外，独占锁也是可抢占的。

每个任务可以有不同的锁定优先级。优先级较高的任务锁可以抢占优先级较低的任务锁。锁定抢占基于*乐观锁定*。锁定被抢占时，不会立即通知所有者任务。相反，当所有者任务尝试再次获取相同的锁时会通知它。（请注意，除非锁被抢占，否则锁获取是幂等的。）通常，任务不竞争获取锁，因为它们通常针对不同的数据源或间隔。

将数据写入dataSource的任务必须获取目标间隔的独占锁。请注意，排他锁仍然是可抢占的。也就是说，除非他们在关键部分*发布片段*，否则它们也能够被更高优先级的锁定所抢占。一旦发布片段完成，这些锁将再次成为可抢占的。

任务不需要显式释放锁，它们在任务完成时释放。如果他们愿意，任务可能会提前释放锁。通过使用UUID或创建任务的时间戳命名它们，任务ID是唯一的。任务也是“任务组”的一部分，“任务组”是一组可以共享间隔锁的任务。

## 优先

Druid的索引任务使用锁来获取原子数据。获取每个锁以用于dataSource和间隔的组合。一旦任务获得锁定，它就可以为dataSource写入数据以及获取锁定的间隔，除非锁定被释放或被抢占。请参阅[下面的锁定部分](http://druid.io/docs/0.12.3/ingestion/locking-and-priority.html#locking)

每个任务都有一个用于锁定获取的优先级。如果优先级较高的任务锁尝试获取相同的dataSource和interval，则可以抢占优先级较低的任务锁。如果任务的某些锁被抢占，则抢占任务的行为取决于任务实现。通常，如果被抢占，大多数任务都会以失败告终。

任务可以根据其类型具有不同的默认优先级。以下是默认优先级列表。数字越大，优先级越高。

| 任务类型           | 默认优先级 |
| ------------------ | ---------- |
| 实时索引任务       | 75         |
| 批量索引任务       | 50         |
| 合并/追加/压缩任务 | 25         |
| 其他任务           | 0          |

您可以通过在任务上下文中设置优先级来覆盖任务优先级，如下所示。

```json
"context" : {
  "priority" : 100
}
```

## 任务背景

任务上下文用于各种任务配置参数。以下参数适用于所有任务类型。

| 属性            | 默认                                                         | 描述                                                         |
| --------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| taskLockTimeout | 300000                                                       | 任务锁定超时（以毫秒为单位）。有关更多详细信息，请参阅[锁定](http://druid.io/docs/0.12.3/ingestion/locking-and-priority.html#locking)。 |
| 优先            | 根据任务类型不同。见[优先权](http://druid.io/docs/0.12.3/ingestion/locking-and-priority.html#priority)。 | 任务优先级                                                   |

当任务获取锁时，它通过HTTP发送请求并等待，直到它收到包含锁获取结果的响应。因此，如果`taskLockTimeout`大于`druid.server.http.maxIdleTime`的霸主，则会发生HTTP超时错误。