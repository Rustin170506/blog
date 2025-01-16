---
title: '给 TiCDC 接上一个 unbound 的 channel'
date: 2022-07-10
description: '在 TiCDC Sink 组件中接入 chann'
tags: ["Go", "TiCDC"]
categories: ["Go"]
---

最近在改造 [TiCDC] 的 [Sink] 组件时需要为 [MQ] 类型的 Sink 接上一个 unbound 的 [channel]。
在搜索过程中发现了一个项目叫做 [chann]，它是一个接口统一并且支持范型的 channel。

虽然这个库看上去实现很简单，但是我在实际使用过程中并不是很顺利。下面我就介绍一下我在使用该库时遇到的问题和进行的思考。

此博客在 [GitHub](https://github.com/Rustin170506/rustin.me) 上公开发布。 如果您有任何问题或疑问，请在此处打开一个 [issue](https://github.com/Rustin170506/rustin.me/issues)。

## 简介
该库由 golang 社区非常活跃的 [changkun] 编写，托管在他组建的 [golang-design] 组织下。

它提供了统一的接口来创建不同类型的 channel，并且支持范型：
```go
ch := chann.New[int]()                  // 无界限, 无容量限制
ch := chann.New[func()](chann.Cap(0))   // 没有缓存, 容量为 0
ch := chann.New[string](chann.Cap(100)) // 有缓存，容量为 100
```

它的发送和接收操作和原生 channel 一致：
```go
ch.In() <- 42
println(<-ch.Out()) // 42
```

它的关闭接口为：
```go
ch.Close()
```

从接口来看，它几乎能无缝的接入到我当前的项目当中，这也是我选择它的原因。

## 源码阅读
在开始分析我遇到的问题之前，我们需要先深入阅读一下源码。它的源码不是很长，并且我主要是用的是无界限的 chann，所以下面就着重看一下无界限的 chann 的源码。

```go
type Chann[T any] struct {
	in, out chan T
	close   chan struct{}
	cfg     *config
	q       []T
}
```

Chann 是一个范型结构体，它里面维护了 `in` 和 `out` channel，这就是我们能使用原生 channel 语法操作 chann 的入口和出口。

另外一个比较关键的字段是 `q`，它将负责存储 chann 的缓存。

它的构造方法：
```go
type config struct {
	typ      chanType
	len, cap int64
}

type Opt func(*config)

func New[T any](opts ...Opt) *Chann[T] {
	cfg := &config{
		cap: -1, len: 0,
		typ: unbounded,
	}

	if len(opts) > 1 {
		panic("chann: too many arguments")
	}
	for _, o := range opts {
		o(cfg)
	}
	ch := &Chann[T]{cfg: cfg, close: make(chan struct{})}
	switch ch.cfg.typ {
	case unbuffered:
		ch.in = make(chan T)
		ch.out = ch.in
	case buffered:
		ch.in = make(chan T, ch.cfg.cap)
		ch.out = ch.in
	case unbounded:
		ch.in = make(chan T, 16)
		ch.out = make(chan T, 16)
		go ch.unboundedProcessing()
	}
	return ch
```

它的构造方法抽象出了一个 Opts 可选参数，根据它我们能够统一构造方法，我们不传递 Opts 参数使用默认 config 就表明创建了一个无界限的 channel。**注意：如果我们创建的是无界限的 chann，那么它将启动一个 goroutine 来处理发送和接收数据。同时入口和出口 channel 的缓存长度为 16。**

接着是它的发送和接收接口：
```go
func (ch *Chann[T]) In() chan<- T { return ch.in }

func (ch *Chann[T]) Out() <-chan T { return ch.out }
```
在这两个接口中我们就是直接返回了 `in` 和 `out` channel，这样我们就能直接使用 go 原生的 channel 语法来操作 chann。

最后，我们来看一下它的核心逻辑，如何处理数据的发送和接收：
```go
func (ch *Chann[T]) unboundedProcessing() {
	var nilT T

	ch.q = make([]T, 0, 1<<10)
	for {
		select {
		case e, ok := <-ch.in:
			if !ok {
				panic("chann: send-only channel ch.In() closed unexpectedly")
			}
			atomic.AddInt64(&ch.cfg.len, 1)
			ch.q = append(ch.q, e)
		case <-ch.close:
			ch.unboundedTerminate()
			return
		}

		for len(ch.q) > 0 {
			select {
			case ch.out <- ch.q[0]:
				atomic.AddInt64(&ch.cfg.len, -1)
				ch.q[0] = nilT
				ch.q = ch.q[1:]
			case e, ok := <-ch.in:
				if !ok {
					panic("chann: send-only channel ch.In() closed unexpectedly")
				}
				atomic.AddInt64(&ch.cfg.len, 1)
				ch.q = append(ch.q, e)
			case <-ch.close:
				ch.unboundedTerminate()
				return
			}
		}
		if cap(ch.q) < 1<<5 {
			ch.q = make([]T, 0, 1<<10)
		}
	}
}

func (ch *Chann[T]) unboundedTerminate() {
	var nilT T

	close(ch.in)
	for e := range ch.in {
		ch.q = append(ch.q, e)
	}
	for len(ch.q) > 0 {
		select {
		case ch.out <- ch.q[0]:
		default:
		}
		ch.q[0] = nilT
		ch.q = ch.q[1:]
	}
	close(ch.out)
	close(ch.close)
}
```

它的主逻辑就是一个 for 循环：
1. 初始化 `q`，初始容量为 1024
2. 启动 for 循环，
3. 进行 select 操作，如果有数据可以接收，则存入 `q` 缓存中，否则如果是 close 则进行关闭操作并退出
4. 如果当前缓存中有数据，则启动内层循环
5. 进行 select 操作，尝试发送缓存中的第一个元素，并且删除它。同时也会尝试接收新的数据，如果有新数据则存入 `q` 缓存中。另外，如果是 close 则进行关闭操作并退出
6. 如果当前缓存中的数据已经全部发送完毕，则退出内层循环并且判断是否需要扩容，然后进入下一次主循环

它的关闭流程：
1. 关闭入口 channel，不再接收新的数据
2. 将当前入口 channel 中的数据全部缓存起来等待处理，尝试优雅的停机
3. 如果缓存中有数据的话，则启动 for 循环不断的将数据发送给出口 channel
4. 最终所有数据都处理完毕，则关闭剩余 channel

以上就是它的核心实现，总的来说还是非常清晰和简单的。下面我们就来看看我在使用过程中遇到的问题！

## 一个 🐛
首先是我在使用过程中遇到的一个 bug，issue 的链接：https://github.com/golang-design/chann/issues/3

我和同事在使用 chann 过程中发现因为关闭操作不当会导致 cpu spin。我们在上面看的代码是目前最新的代码，但是在我报告这个 issue 之前它的 close 操作的核心逻辑是这样的：
```go
for len(q) > 0 {
    select {
		case ch.out <- q[0]:
			    q[0] = nilT // de-reference earlier to help GC
				q = q[1:]
		default:
	}
}
```
请注意它和上面的操作完全是两个语义：
```go
for len(q) > 0 {
    select {
	    case ch.out <- q[0]:
		default:
	}
	q[0] = nilT // de-reference earlier to help GC
    q = q[1:]
}
```

原来的写法，在 close 操作之后如果没有 receiver 继续接收数据，则 for 循环会一直进入 default 分支进行死循环，导致出现 cpu spin。

而现在的做法是，当没有 receiver 时会进入 default 分支并且直接删除该数据，这样就能防止出现死循环导致 cpu spin。

目前这样的做法也导致了语义的变化，我们现在实际上已经做不到优雅的停机了，因为很有可能在 default 分支被执行之后消费者丢失部分数据。具体的例子可以查看 [changkun] 的[回复]。

目前这个问题还是没有被完全解决，在我个人来看，我更愿意保持原来的做法让用户来处置剩下的数据，这样语义更加明确。但是我们还是需要删除原来做法中的 default 分支，防止出现死循环。让用户来保证消费完所有剩余数据防止出现 goroutine 泄漏。

但这只是我个人的想法，[changkun] 实际上不太认可这个做法，因为还是存在 goroutine 泄漏的风险。期待我们有更好的方案能彻底解决这个问题。

因为该问题没有被解决，所以我决定将代码 [fork] 到 [ticdc] 仓库中，在这个过程中我又发现了一个不稳定的测试。

## 一个不稳定测试
在 fork 代码的过程中，我发现 `TestNonblockSelectRace` 非常不稳定，在我们的 CI 系统中几乎稳定失败：
```go
func TestNonblockSelectRace(t *testing.T) {
	n := 1000
	if testing.Short() {
		n = 1000
	}
	done := chann.New[bool](chann.Cap(1))
	for i := 0; i < n; i++ {
		c1 := chann.New[int]()
		c2 := chann.New[int]()
		c1.In() <- 1
		go func() {
			runtime.Gosched()
			select {
			case <-c1.Out():
			case <-c2.Out():
			default:
				done.In() <- false
				return
			}
			done.In() <- true
		}()
		c2.In() <- 1
		select {
		case <-c1.Out():
		default:
		}
		if !<-done.Out() {
			t.Fatal("no chan is ready")
		}
	}
}
```

这个测试的语义的保证是，当数据通过 In() 接口被写入之后，我们总是立马就能消费到该数据，但是实际上我们并不能提供这个语义保证，因为数据被写入之后还要等待被写入缓存中，才有可能被消费到。

我做了一些修复降低了这个问题复现的频率，但是因为我们还是无法保证这个语义，所以最终 [PR] 还是没有被接受。

最终 [changkun] 提交了代码明确了我们没有这个语义保证并且修复了这个不稳定测试。
新的语义如下：
> // An unbounded channel is not a buffered channel with infinite capacity,
> and they have different memory model semantics in terms of receiving
> a value: The recipient of a buffered channel is immediately available
> after a send is complete. However, the recipient of an unbounded channel
> may be available within a bounded time frame after a send is complete.

他测试修改的也很巧妙：
```go
func TestNonblockSelectRace(t *testing.T) {
	n := 1000
	done := chann.New[bool](chann.Cap(1))
	for i := 0; i < n; i++ {
		c1 := chann.New[int]()
		c2 := chann.New[int]()

		// The input channel of an unbounded buffer have an internal
		// cache queue. When the input channel and the internal cache
		// queue both gets full, we are certain that once the next send
		// is complete, the out will be available for sure hence the
		// waiting time of a receive is bounded.
		for i := 0; i < internalCacheSize; i++ {
			c1.In() <- 1
		}
		c1.In() <- 1
		go func() {
			select {
			case <-c1.Out():
			case <-c2.Out():
			default:
				done.In() <- false
				return
			}
			done.In() <- true
		}()
		// Same for c2
		for i := 0; i < internalCacheSize; i++ {
			c2.In() <- 1
		}
		c2.In() <- 1
		select {
		case <-c1.Out():
		default:
		}
		if !<-done.Out() {
			t.Fatal("no chan is ready")
		}
	}
}
```

因为我们的缓存队列大小是固定的，所以当我们确保缓存已满之后再写入时，我们总会有足够的时间收到写入的数据。

## 学到的小技巧

在 fork 代码的过程中发现了几个 [changkun] 的代码中很有用的技巧。

1. 在切片中的数据被消费使用之后，将其设置为 nil，并且在容量不足时总是 make 新的切片，这样会帮助 GC 更快回收内存。
2. 创建 `export_test.go` 文件导出测试专用函数，既能访问到内部状态，又不用将单元测试也放在同一个包下导致测试到处都依赖内部状态。
3. 利用 `runtime.GOMAXPROCS` 和 `runtime.NumGoroutine` 来构造特定 goroutine 测试场景和检测 goroutine 泄漏。

### 参考链接

[chann](https://github.com/golang-design/chann)

[ultimate-channel](https://golang.design/research/ultimate-channel/)

[Will gc collect the objects while the array set to nil in Golang?](https://stackoverflow.com/questions/50732829/will-gc-collect-the-objects-while-the-array-set-to-nil-in-golang)


[chann]: https://github.com/golang-design/chann

[ticdc]: https://github.com/pingcap/tiflow

[sink]: https://ticdc-sink.slides.rustin.me/

[mq]: https://en.wikipedia.org/wiki/Message_queue

[channel]: https://go.dev/tour/concurrency/2

[changkun]: https://github.com/changkun

[golang-design]: https://github.com/golang-design

[回复]: https://github.com/golang-design/chann/issues/3#issuecomment-1150189421

[fork]:https://github.com/pingcap/tiflow/pull/5675

[pr]: https://github.com/golang-design/chann/pull/2
