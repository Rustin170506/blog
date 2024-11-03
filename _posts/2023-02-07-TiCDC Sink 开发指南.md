---
title: 'TiCDC Sink 开发指南'
layout: post

categories: post
tags:
- Golang
- Go
- TiCDC
- TiDB
---

我近半年的时间都在做 [TiCDC] [Sink 模块]的改造工作，目前新的 Sink 实现已经成功替换了旧的实现。最近有客户希望通过自己实现 Sink 的方式来接入 TiCDC，所以我想把这段时间的改造和设计经验分享出来，希望能帮助到大家。

此博客在 [GitHub](https://github.com/Rustin170506/rustin.me) 上公开发布。 如果您有任何问题，请在此处打开一个 [issue](https://github.com/Rustin170506/rustin.me/issues)。

> ⚠️ 注意：
> 1. 该指南主要面向开发者，如果您只是想使用 TiCDC，请参阅 [TiCDC 使用文档](https://docs.pingcap.com/zh/tidb/stable/ticdc-overview)。
> 2. 在阅读该指南前，请先阅读 [TiCDC 架构和数据同步链路解析](https://rustin.me/TiCDC-%E6%9E%B6%E6%9E%84%E5%92%8C%E6%95%B0%E6%8D%AE%E5%90%8C%E6%AD%A5%E9%93%BE%E8%B7%AF%E8%A7%A3%E6%9E%90/)了解 TiCDC 的基本架构和数据同步流程。

## 基本概念

> 可以先简单浏览这些子组件概念，后面会有详细的介绍。

- Sink：TiCDC 的 Sink 模块负责将 TiCDC 的数据变更输出到外部系统中。目前 TiCDC 支持输出到 MySQL、TiDB、Kafka、S3 等外部系统中。
- [Table Sink]：负责将 TiCDC 的数据变更按照表为单位进行聚合，然后输出到外部系统中。
- [Event Sink]：负责与外部系统进行交互，将 TiCDC 的数据变更编码后输出到外部系统中。这里的 Event 主要指的是 TiCDC 的数据变更事件，比如 Insert、Update、Delete 等。
- [MQ Event Sink]：负责将 TiCDC 的数据变更输出到 Message Queue 中。MQ Sink 会将数据变更事件编码为 MQ 消息，然后输出到 MQ 中。目前 TiCDC 支持输出到 Kafka 中。
- [Txn Event Sink]：负责将 TiCDC 的数据变更按照事务为单位进行聚合，然后输出到外部系统中。目前 TiCDC 支持输出到 MySQL、TiDB 中。
- [DDL Sink]：负责将 TiCDC 的 DDL 语句输出到外部系统中。目前 TiCDC 支持输出到 Kafka、MySQL、TiDB 中。
- [MQ DDL Sink]：负责将 TiCDC 接受到的 DDL 语句输出到 Kafka 中。
- [Txn DDL Sink]：，负责将 TiCDC 接受到的 DDL 语句输出到 MySQL、TiDB 中。

## 基本架构

我们可以将 TiCDC 接收到的数据分为两类：
- DML：TiCDC 接收到的数据变更事件，比如 Insert、Update、Delete 等。
- DDL：TiCDC 接收到的 DDL 语句。

Sink 模块也根据上述不同的数据类型抽象出了不同的 Sink 子模块，分别是 [Event Sink]、[DDL Sink]。Event Sink 负责将过滤和聚合后的 DML 数据输出到外部系统中，DDL Sink 负责将过滤后的 DDL 数据输出到外部系统中。

DDL Sink 很容易理解，因为它就是简单的将收到每张表的 DDL 语句编码后输出到外部系统。而 Event Sink 则更加复杂，我们会接收到大量不同表的变更数据，但是 TiCDC 需要按照表为单位进行数据同步。所以我们又引入了 [Table Sink]，它负责将收到的数据按照表进行聚合，然后输出到 Event Sink 中。

我们可以将 TiCDC 的 Sink 模块抽象为下面这个图：

[![](https://www.plantuml.com/plantuml/png/ROwn3e8m48RtFiN9Q08FG1mOO1CJ0eFhYPVOs5P2MoSVtjCsX0Qsotr_ll-lhCFPUQt4mJr84qmAfH6ZGcjXw04jP0FU544lpJEBe0cWUPDn2MYxGDeEjd2uNg9mHcDnTF9bafZWmcEU__GbU4j2y7Nw5CNVMuBKaoBD-UNFoXJ4ghe-Xoe-edm1guqxT6_aAYVu3DrHbRGdSBEj8dFtMdq1)](https://www.plantuml.com/plantuml/uml/ROwn3e8m48RtFiN9Q08FG1mOO1CJ0eFhYPVOs5P2MoSVtjCsX0Qsotr_ll-lhCFPUQt4mJr84qmAfH6ZGcjXw04jP0FU544lpJEBe0cWUPDn2MYxGDeEjd2uNg9mHcDnTF9bafZWmcEU__GbU4j2y7Nw5CNVMuBKaoBD-UNFoXJ4ghe-Xoe-edm1guqxT6_aAYVu3DrHbRGdSBEj8dFtMdq1)

有了这个基本的架构，我们就可以看看数据是如何在各个子模块之间流动的了。

## 数据流程

数据同步流程也可以根据数据类型分为两部分，一部分是 DML 数据，另一部分是 DDL 数据。

### DML 数据
TiCDC 从 TiKV 接受到变更数据后，会对数据进行排序，但是整个排序过程中数据都是所有表的数据放在一起进行排序的。排序完成后我们还需要以表为单位将数据进行分发，这个过程就是 Table Sink 负责的。所以其他组件跟 Sink 模块的交互都是通过调用 Table Sink 的接口来完成的。

这个过程中 Table Sink 可以理解成一个缓冲区，它会将收到的数据按照表进行缓存，但是并不会真实的将数据写入外部系统。与外部系统的交互是通过 Event Sink 来完成的，通过这样的抽象，多个 Table Sink 可以共享一个 Event Sink，我们可以在底层并发的进行数据写入。另外，我们也能共用一些公共的资源，比如数据库连接池，Kafka 的生产者等等。

下面是 DML 数据在 [MQ Event Sink] 模块中流转的时序图：

[![](https://www.plantuml.com/plantuml/png/VT2n2eCm40RWFKznTNSmeqEn5QSWL1Gwd_5AGsAK6FlyZQ6Kwj0bm_z_kOCh5e_EhwDX9_-aaM0sg2oRGwYacj5wwDeCS86amzuGjChgB3a0VW1y3-gcQgEe6wXUP7r4UoCY4FZG25StQN89Ozlgz1p_vo3H6BWxvIdEM4BB_xHRlDKYXvkRXbKI4v1-_QKKaSGexFbCACFJezJPRyaF9MS5sI4SxGq0)](https://www.plantuml.com/plantuml/uml/VT2n2eCm40RWFKznTNSmeqEn5QSWL1Gwd_5AGsAK6FlyZQ6Kwj0bm_z_kOCh5e_EhwDX9_-aaM0sg2oRGwYacj5wwDeCS86amzuGjChgB3a0VW1y3-gcQgEe6wXUP7r4UoCY4FZG25StQN89Ozlgz1p_vo3H6BWxvIdEM4BB_xHRlDKYXvkRXbKI4v1-_QKKaSGexFbCACFJezI_7Jzs1TaXdEmD)

在上图中 Table Sink1 和 Table Sink2 通过调用 Event Sink 的 `WriteEvents` 将数据异步的写入到 Kafka 中，在等到写入的消息收到 ACK 之后，再通过调用每个事件的 Callback 来通知 Table Sink 数据已经写入成功。**整个数据写入过程都是异步的，这样可以提高数据写入的吞吐量。**


### DDL 数据

TiCDC 从 TiKV 接受到 DDL 变更数据后，会将数据直接发送到具体的 [DDL Sink] 实现中，这个过程是同步的，也就是说 TiCDC 会等待 DDL Sink 返回成功之后才会继续处理后续的 DDL 变更数据。在写入 DDL 的时候我们并没有使用 Table Sink，因为 DDL 数据是全局共用且有序的，所以我们可以直接将数据发送到 DDL Sink 中。

下面是 DDL 数据在 MQ DDL Sink 模块中流转的时序图：

[![](https://www.plantuml.com/plantuml/png/LSungiCm40JGVayntxqluE9ZWZl54ECJBD862yah8Sj5Slg894swPUSDp7XKBlNS8_tLJNP1ZkoLSdjwwpDhnRnCqtK57-Zc1Ut6wZLqFyQyOyFtmBYK5AHqHDzY_mypm7ACM1zgjvBKFyNXLf8xhP11tyW73qY1Hb7N5hq0)](https://www.plantuml.com/plantuml/uml/LSungiCm40JGVayntxqluE9ZWZl54ECJBD862yah8Sj5Slg894swPUSDp7XKBlNS8_tLJNP1ZkoLSdjwwpDhnRnCqtK57-Zc1Ut6wZLqFyQyOyFtmBYK5AHqHDzY_mypm7ACM1zgjvBKFyNXLf8xhP11tyW73qY1Hb7N5hq0)

## Table Sink

在 Sink 模块中，Table Sink 是一个抽象的接口：

```golang
// https://github.com/pingcap/tiflow/blob/master/cdc/sinkv2/tablesink/table_sink.go
// 用于将数据以表为单位进行缓存。
type TableSink interface {
	// AppendRowChangedEvents 将行变更事件追加到 Table Sink 中。
	// 注意：此方法不是线程安全的，请不要并发调用。
	AppendRowChangedEvents(rows ...*model.RowChangedEvent)
	// UpdateResolvedTs 将聚合完成的数据发送到 Event Sink 中。
	// 注意：此方法是异步的且不是线程安全的。
	UpdateResolvedTs(resolvedTs model.ResolvedTs) error
	// GetCheckpointTs 返回 Table Sink 中的 CheckpointTs。
	// 注意：此方法是线程安全的。
	GetCheckpointTs() model.ResolvedTs
	// Close 关闭 Table Sink 并释放资源。
	// 注意：我们需要保证这个方法是可取消或者可中断的。
	Close(ctx context.Context)
}
```

它最重要的两个方法是 `AppendRowChangedEvents` 和 `UpdateResolvedTs`，前者用于将行变更事件追加到 Table Sink 的缓存中，后者用于将聚合完的数据发送到 Event Sink 中。在 TiCDC 中，我们实现了一个基于内存的 Table Sink，它会将行变更事件聚合到内存中。

因为针对的外部系统的不同，我们需要实现不同的聚合策略。所以我们为 Table Sink 的实现添加了一个范型参数 `E`，用于指定聚合策略：

```golang
// https://github.com/pingcap/tiflow/blob/master/cdc/sinkv2/tablesink/table_sink.go
type EventTableSink[E eventsink.TableEvent] struct {
	...
	// 就是具体的 Event Sink，比如 MQ Event Sink。
	backendSink     eventsink.EventSink[E]
	...
	// 用于实现不同的聚合策略。
	eventAppender   eventsink.Appender[E]
	// 注意：数据是按照 CommitTs 排序的。
	eventBuffer []E
	...
}
```

可以看到范型参数 `E` 的类型为 `eventsink.TableEvent`，它是一个非常简单的接口：

```golang
// https://github.com/pingcap/tiflow/blob/master/cdc/sinkv2/eventsink/event.go
type TableEvent interface {
	// GetCommitTs 返回事件的 CommitTs。
	GetCommitTs() uint64
}
```

通过这个接口抽象，任何可以获取 CommitTs 的事件都可以作为 Table Sink 的缓存对象。在 TiCDC 中，我们实现了两种聚合策略：

- `RowChangedEvent`：用于单行变更，比如 [MQ Event Sink] 就是将行变更一条一条发送到 Kafka 中。
- `SingleTableTxn`：用于单表事务，比如 [Txn Event Sink] 就是以事务为单位提交到 MySQL 中。

还记得我们在数据流程中提到 DML 数据的写入是异步的吗？所以我们需要为每个 Event 添加一个 Callback，用于在数据写入完成后通知 Table Sink：

```golang
// https://github.com/pingcap/tiflow/blob/master/cdc/sinkv2/eventsink/event.go
type CallbackFunc func()

type CallbackableEvent[E TableEvent] struct {
	Event     E
	Callback  CallbackFunc
	...
}

// RowChangeCallbackableEvent 是行变更事件，它可以被回调。
type RowChangeCallbackableEvent = CallbackableEvent[*model.RowChangedEvent]

// TxnCallbackableEvent 是单表事务事件，它可以被回调。
type TxnCallbackableEvent = CallbackableEvent[*model.SingleTableTxn]
```

有了这两种不同的可以回调的 Event，我们就可以实现具体的聚合策略了。为了复用代码，我们将聚合策略抽象为一个接口：

```golang
// https://github.com/pingcap/tiflow/blob/master/cdc/sinkv2/eventsink/event_appender.go
type Appender[E TableEvent] interface {
	// Append 添加一批行变更事件到缓存中。
	Append(buffer []E, rows ...*model.RowChangedEvent) []E
}
```

**这样我们只需要实现 `Append` 方法，就可以实现不同的聚合策略了。**

对于 `RowChangeCallbackableEvent` 来说，我们并没有实际上的聚合操作，只是将行变更事件**顺序**追加到当前的缓存中：

```golang
// https://github.com/pingcap/tiflow/blob/master/cdc/sinkv2/eventsink/event_appender.go
func (r *RowChangeEventAppender) Append(
	buffer []*model.RowChangedEvent,
	rows ...*model.RowChangedEvent,
) []*model.RowChangedEvent {
	return append(buffer, rows...)
}
```

对于 `TxnCallbackableEvent` 来说，我们需要将行变更事件聚合到一个事务中：

```golang
// https://github.com/pingcap/tiflow/blob/master/cdc/sinkv2/eventsink/event_appender.go
func (t *TxnEventAppender) Append(
	buffer []*model.SingleTableTxn,
	rows ...*model.RowChangedEvent,
) []*model.SingleTableTxn {
	for _, row := range rows {
		// 这意味我们目前还没有任何事务，所以我们需要创建一个新的事务。
		if len(buffer) == 0 {
			txn := &model.SingleTableTxn{
				StartTs:   row.StartTs,
				CommitTs:  row.CommitTs,
				Table:     row.Table,
				TableInfo: row.TableInfo,
			}
			txn.Append(row)
			buffer = append(buffer, txn)
			continue
		}

		lastTxn := buffer[len(buffer)-1]

		lastCommitTs := lastTxn.GetCommitTs()
		...
		// 使用 StartTs 来判断是否是同一个事务。如果不是同一个事务，我们需要创建一个新的事务。
		if row.SplitTxn || lastTxn.StartTs != row.StartTs {
			buffer = append(buffer, &model.SingleTableTxn{
				StartTs:   row.StartTs,
				CommitTs:  row.CommitTs,
				Table:     row.Table,
				TableInfo: row.TableInfo,
			})
		}

		buffer[len(buffer)-1].Append(row)
	}

	return buffer
}
```

在上面的代码中，我们将 StartTs 作为界限，将行变更事件聚合到一个事务中。**在 TiDB 中两个事务的 CommitTs 可能相同，但是 StartTs 一定不同，所以我们可以通过 StartTs 来判断行变更是否属于同一个事务。**

有了不同聚合策略的 `Appender`，我们就可以实现具体的 Table Sink 了。首先是 `AppendRowChangedEvents` 方法，得益于 `Appender` 的抽象，我们的具体实现可以非常简单：

```golang
// https://github.com/pingcap/tiflow/blob/master/cdc/sinkv2/tablesink/table_sink.go
func (e *EventTableSink[E]) AppendRowChangedEvents(rows ...*model.RowChangedEvent) {
	e.eventBuffer = e.eventAppender.Append(e.eventBuffer, rows...)
}
```

其次是 `UpdateResolvedTs` 方法，它会将当前的缓存中的所有 Event 写入到具体的 Event Sink 中：

```golang
// https://github.com/pingcap/tiflow/blob/master/cdc/sinkv2/tablesink/table_sink.go
func (e *EventTableSink[E]) UpdateResolvedTs(resolvedTs model.ResolvedTs) error {
	...
	// 从缓存中找到第一个大于 resolvedTs 的数据。
	i := sort.Search(len(e.eventBuffer), func(i int) bool {
		return e.eventBuffer[i].GetCommitTs() > resolvedTs.Ts
	})
	...
	// 将该数据之前的所有数据取出来。
	resolvedEvents := e.eventBuffer[:i]

	...

	// 为每一个取出来的数据创建一个 CallbackableEvent。
	resolvedCallbackableEvents := make([]*eventsink.CallbackableEvent[E], 0, len(resolvedEvents))
	for _, ev := range resolvedEvents {
		ce := &eventsink.CallbackableEvent[E]{
			Event:     ev,
			Callback:  e.progressTracker.addEvent(),
			SinkState: &e.state,
		}
		resolvedCallbackableEvents = append(resolvedCallbackableEvents, ce)
	}

	...

	// 将取出来的数据写入到下游。
	return e.backendSink.WriteEvents(resolvedCallbackableEvents...)
}
```

可以看到在 `UpdateResolvedTs` 方法中，我们将缓存中的数据以 `ResolvedTs` 为界限取出来，然后将其转换为 `CallbackableEvent`，最后将其写入到具体的 backendSink 中。**需要注意的是，我们能这样做的前提是数据在之前的模块中已经经过了排序，这样我们才能保证数据的顺序性。**

以上就是 Table Sink 的核心实现，只要实现不同的 `Appender`，我们就可以实现不同的 Table Sink。目前 TiCDC 主要支持的就是上述的两种聚合策略，你可以根据自己的需求来选择不同的聚合策略。**一般来说，你不需要自己实现 Table Sink，目前的两种聚合策略已经能满足大部分的需求。**

## Event Sink

在 TiCDC 中，[Event Sink] 是真实与外部系统交互的模块，它的主要职责是将 Table Sink 中的数据写入到外部系统中。它的主要接口如下：

```golang
// https://github.com/pingcap/tiflow/blob/master/cdc/sinkv2/eventsink/event_sink.go
type EventSink[E TableEvent] interface {
	// WriteEvents 将数据写入到外部系统中。
	// 注意：这是一个异步且线程安全的方法。
	WriteEvents(events ...*CallbackableEvent[E]) error
	// Close 关闭 Event Sink。
	// 注意：我们需要保证这个方法是可取消或者可中断的。
	Close() error
}
```

这个接口的实现是与具体的外部系统相关的，我们以 [MQ Event Sink] 为例来看一下它的实现：

```golang
// https://github.com/pingcap/tiflow/blob/master/cdc/sinkv2/eventsink/mq/mq_dml_sink.go
type dmlSink struct {
	...
	worker *worker
	...
	ctx    context.Context
	cancel context.CancelFunc
}
```
可以看到，它的实现中包含了一个 `worker`，这个 `worker` 用来处理写入到外部系统的数据。它的主要职责是将写入的数据进行编码，把 TiDB 的数据类型转化为 MQ 系统的数据类型，然后异步的将数据写入到 MQ 系统中。我们在这里就不展开讲解了，因为它的实现与具体的外部系统相关，你可以在 [Sink 模块] 中找到它的具体实现。

我们来看一下 `WriteEvents` 方法的实现：

```golang
// https://github.com/pingcap/tiflow/blob/master/cdc/sinkv2/eventsink/mq/mq_dml_sink.go
func (s *dmlSink) WriteEvents(rows ...*eventsink.RowChangeCallbackableEvent) error {
	for _, row := range rows {
		...
		// 将数据通过 channel 发送到 worker 中。
		s.worker.msgChan.In() <- mqEvent{
			...
			rowEvent: row,
		}
	}

	return nil
}
```

它的实现非常简单，就是将数据通过 channel 发送到 worker 中，然后 worker 封装了具体的写入逻辑，主要是将数据进行编码后调用外部系统的 API 进行写入。例如：将 TiDB 的数据编码为一条 Kafka 消息，然后调用 Kafka 生产者的 Produce 方法进行写入。

**我们可以注意到，这里的 `rows` 都是从 Table Sink 中下发的可以回调的数据，在 worker 写入成功后，我们就可以通过回调的方式来通知 Table Sink 数据已经写入成功。**

## DDL Sink

在 TiCDC 中，DDL Event 比较特殊，因为 DDL 是全局共用的，所以我们只需要有一个 DDL Sink 按照顺序将 DDL 写入到外部系统中即可。它的主要接口如下：

```golang
// https://github.com/pingcap/tiflow/blob/master/cdc/sinkv2/ddlsink/ddl_sink.go
type DDLEventSink interface {
	// WriteDDLEvent 将 DDL 写入到外部系统中。
	// 注意：这是一个同步且线程安全的方法。
	WriteDDLEvent(ctx context.Context, ddl *model.DDLEvent) error
	// WriteCheckpointTs 将 CheckpointTs 写入到外部系统中。
	// 注意：这是一个同步且线程安全的方法。
	// 目前只有 MQ DDL Sink 会实现这个方法。
	WriteCheckpointTs(ctx context.Context, ts uint64, tables []*model.TableInfo) error
	// Close 关闭 DDL Sink 和释放相关资源。
	Close() error
}
```

它的核心方法是 `WriteDDLEvent`，我们将 DDL Event 传入之后，它会将 DDL Event 进行编码，然后写入到外部系统中。例如：将 TiDB 的 DDL Event 编码为一条 Kafka 消息，然后调用 Kafka 生产者的 Produce 方法进行写入。**注意，这个方法的实现需要是同步的，因为 DDL 数量不会很多，所以我们可以将它的实现设置为同步的，这样可以避免一些并发问题，让实现变得简单。**

值得一提的是，MQ DDL Sink 会实现 `WriteCheckpointTs` 方法，它会将 CheckpointTs 写入到 MQ 系统中。它会作为一个特殊的消息告诉下游系统当前的 CheckpointTs，我们可以把它理解成 MQ 系统中的 Watermark。这样，我们在消费 MQ 系统中的消息时，就可以通过 CheckpointTs 来进行过滤，排序等操作。

## Log Sink 示例

介绍了上面的各种 Sink 概念和实现之后，我们可以动手自己实现一个简单的 Log Sink 来了解如何为 TiCDC 添加一个新的 Sink。我们可以假设 Log Sink 的主要职责就是将所有的数据输出到日志中。我们调用 `log.Info()` 写入数据即可。

首先，我们需要确认聚合策略，根据不同的聚合策略我们可以选用不同的 Table Sink 实现。因为我们的 Log Sink 只是简单的将数据写入到日志中，所以我们可以选择 `RowChangeEventAppender` 作为我们的 `Appender`。这样每一行数据都会被写入到日志中。

接下来，我们只需要实现一个具体的 Event Sink 和一个 DDL Sink 即可。首先是 Event Sink：

```golang

// 断言 LogSink 实现了 EventSink 接口。
var _ eventsink.EventSink[*model.RowChangedEvent] = (*LogSink)(nil)

type LogSink struct{}

// New 创建一个新的 LogSink。
func New() *LogSink {
	return &LogSink{}
}

func (s *LogSink) WriteEvents(rows ...*eventsink.CallbackableEvent[*model.RowChangedEvent]) error {
	for _, row := range rows {
		log.Info("LogSink: WriteEvents", zap.Any("row", row.Event))
		// 不要忘记调用 Callback 方法来通知 Table Sink 该 Event 已经被处理。
		row.Callback()
	}

	return nil
}

func (s *LogSink) Close() error {
	return nil
}
```

我们在实现 `WriteEvents` 方法时，只需要将每一行数据写入到日志中即可。在 此外，我们还需要调用 `Callback` 方法来通知 Table Sink 该 Event 已经被处理。**注意，所有的 `Callback` 方法是必须调用的，否则 Table Sink 会一直等待该 Event 被处理，这样会导致 TiCDC 数据同步卡住。**

另外，我们可以注意到我在第一行代码中做了一个断言，这是为了确保我们的 Log Sink 实现了 `EventSink` 接口。这样，我们在编译时就可以发现一些错误。建议大家在实现 Sink 时都做这样的断言。

接下来是 DDL Sink：

```golang
// 断言 LogSink 实现了 DDLEventSink 接口。
var _ ddlsink.DDLEventSink = (*LogSink)(nil)

type LogSink struct{}

// New create a black hole DDL sink.
func New() *LogSink {
	return &LogSink{}
}

func (d *LogSink) WriteDDLEvent(ctx context.Context,
	ddl *model.DDLEvent,
) error {
	log.Info("LogSink: DDL Event", zap.Any("ddl", ddl))
	return nil
}

func (d *LogSink) WriteCheckpointTs(ctx context.Context,
	ts uint64, tables []*model.TableInfo,
) error {
	log.Info("LogSink: Checkpoint Ts Event", zap.Uint64("ts", ts), zap.Any("tables", tables))
	return nil
}

// Close do nothing.
func (d *LogSink) Close() error {
	return nil
}
```

DDL Sink 的实现非常简单，我们只需要将 DDL Event 写入到日志中即可。

有了具体的 Event Sink 和 DDL Sink 之后，我们就可以尝试将其接入到 TiCDC 中了。在接入之前，**我们需要了解一下 TiCDC 目前是如何构造 Sink 的。
因为 golang 范型的限制，我们无法直接将一个带范型的 Event Sink 传递给 Table Sink。所以我们构建了一个[构造工厂]来解决这个问题：**

```golang
// https://github.com/pingcap/tiflow/blob/master/cdc/sinkv2/eventsink/factory/factory.go
type SinkFactory struct {
	sinkType sink.Type
	rowSink  eventsink.EventSink[*model.RowChangedEvent]
	txnSink  eventsink.EventSink[*model.SingleTableTxn]
}

// New 根据协议类型创建一个 SinkFactory。
func New(ctx context.Context,
	sinkURIStr string,
	cfg *config.ReplicaConfig,
	errCh chan error,
) (*SinkFactory, error) {
	sinkURI, err := config.GetSinkURIAndAdjustConfigWithSinkURI(sinkURIStr, cfg)
	if err != nil {
		return nil, err
	}

	s := &SinkFactory{}
	schema := strings.ToLower(sinkURI.Scheme)
	switch schema {
	case sink.MySQLScheme, sink.MySQLSSLScheme, sink.TiDBScheme, sink.TiDBSSLScheme:
		txnSink, err := txn.NewMySQLSink(ctx, sinkURI, cfg, errCh, txn.DefaultConflictDetectorSlots)
		if err != nil {
			return nil, err
		}
		s.txnSink = txnSink
		s.sinkType = sink.TxnSink
	case sink.KafkaScheme, sink.KafkaSSLScheme:
		mqs, err := mq.NewKafkaDMLSink(ctx, sinkURI, cfg, errCh,
			kafka.NewSaramaAdminClient, kafka.NewSaramaClient, dmlproducer.NewKafkaDMLProducer)
		if err != nil {
			return nil, err
		}
		s.rowSink = mqs
		s.sinkType = sink.RowSink
	...
	default:
		return nil,
			cerror.ErrSinkURIInvalid.GenWithStack("the sink scheme (%s) is not supported", schema)
	}

	return s, nil
}

// CreateTableSink 创建一个 Table Sink，并且传入 Event Sink。
func (s *SinkFactory) CreateTableSink(
	changefeedID model.ChangeFeedID, span tablepb.Span, totalRowsCounter prometheus.Counter,
) tablesink.TableSink {
	switch s.sinkType {
	case sink.RowSink:
		// 我们需要在这里显式地指定 Event Sink 的类型。否则 golang 无法推断出该类型。
		return tablesink.New[*model.RowChangedEvent](changefeedID, span,
			s.rowSink, &eventsink.RowChangeEventAppender{}, totalRowsCounter)
	case sink.TxnSink:
		return tablesink.New[*model.SingleTableTxn](changefeedID, span,
			s.txnSink, &eventsink.TxnEventAppender{}, totalRowsCounter)
	default:
		panic("unknown sink type")
	}
}
```

首先，我们通过 `New` 函数根据协议类型创建一个 SinkFactory。这个过程中我们会创建一个 Event Sink 的具体实例。比如我们新增了 Log Sink，那么我们就可以在这里创建一个 Log Sink 的实例：

```diff
// https://github.com/pingcap/tiflow/blob/master/cdc/sinkv2/eventsink/factory/factory.go
func New(ctx context.Context,
	sinkURIStr string,
	cfg *config.ReplicaConfig,
	errCh chan error,
) (*SinkFactory, error) {
	sinkURI, err := config.GetSinkURIAndAdjustConfigWithSinkURI(sinkURIStr, cfg)
	if err != nil {
		return nil, err
	}

	s := &SinkFactory{}
	schema := strings.ToLower(sinkURI.Scheme)
	switch schema {
	...
	...
+	case sink.LogScheme:
+		s.rowSink = logsink.New()
+		// 根据我们上面选择的聚合策略，我们选择 RowSink。
+		s.sinkType = sink.RowSink
	default:
		return nil,
			cerror.ErrSinkURIInvalid.GenWithStack("the sink scheme (%s) is not supported", schema)
	}

	return s, nil
}
```

在这里我们新增了一个 `LogScheme`，假设它为 `log://`，那么当用户传入 `--sink-uri="log://"` 时，我们就会创建一个 Log Sink 的实例。**在创建过程中我们使用了 `sink.RowSink` 作为 Table Sink 的类型，这是因为我们选择了以行为单位的聚合策略。**

最后，我们需要关注一下 `CreateTableSink` 函数，这个函数会创建一个 Table Sink，并且传入 Event Sink。可以看到我们在这里根据 `sinkType` 指定的范型参数为 `*model.RowChangedEvent` 这样就实现了 Table Sink 和 Event Sink 的组合。

这样我们就完整地实现了一个新的 Sink。接下来只需要指定 `--sink-uri="log://"` 即可使用我们的 Log Sink。

## 总结

本文我介绍了 TiCDC 的 Sink 模块，我们了解了 Sink 详细设计，以及如何实现一个新的 Sink。但是要实现一个可用的 Sink，我们还需要考虑更多的问题，比如：

- 数据写入的效率
- 数据编码效率
- 数据写入失败的处理
- 参数和配置的处理
- 单元测试的编写
- 集成测试的编写
- 使用文档的编写
- 模块的可观测性
- 输出的数据如何被消费和使用
- 输出的数据的正确性校验

等等。**所以如果你真的希望自己动手实现一个生产级别的 Sink，那么你可以参考我们目前已经 GA 的 Sink 实现来实现你自己的 Sink。**如果你在实现过程中遇到了问题，欢迎在 TiCDC 的 Issue Tracker 中提出问题我们一起讨论和解决问题。

[sink 模块]: https://github.com/pingcap/tiflow/tree/master/cdc/sinkv2
[ticdc]: https://docs.pingcap.com/tidb/dev/ticdc-overview/
[table sink]: https://github.com/pingcap/tiflow/tree/master/cdc/sinkv2/tablesink
[event sink]: https://github.com/pingcap/tiflow/tree/master/cdc/sinkv2/eventsink
[mq event sink]: https://github.com/pingcap/tiflow/tree/master/cdc/sinkv2/eventsink/mq
[txn event sink]: https://github.com/pingcap/tiflow/tree/master/cdc/sinkv2/eventsink/txn
[ddl sink]: https://github.com/pingcap/tiflow/tree/master/cdc/sinkv2/ddlsink
[mq ddl sink]: https://github.com/pingcap/tiflow/tree/master/cdc/sinkv2/ddlsink/mq
[txn ddl sink]: https://github.com/pingcap/tiflow/tree/master/cdc/sinkv2/ddlsink/mysql
[构造工厂]: https://github.com/pingcap/tiflow/tree/master/cdc/sinkv2/eventsink/factory
