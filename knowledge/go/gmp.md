# GMP

> 参考
> [从零理解Go语言GMP调度模型（上）](https://www.bilibili.com/video/BV1ioGczjEgA/?share_source=copy_web&vd_source=51c71b6bfa7dec49dc80c06885da3aa7)
> [万字文稿加动画搞懂Go语言gmp调度模型(下)](https://www.bilibili.com/video/BV1x9gMzyEVu)

GMP模型是go语言的实际执行模型。

- G：Goroutine
- M：Machine，指代操作系统线程
- P：Processor，go语言的资源分配和协调器

## 1. 调度是如何进行的？

1. 系统初始化一个P链表，该链表的大小由用户指定或通过默认值设置（一般为CPU核心数）。
2. M创建，并绑定一个g0, 这个g0是调度程序，用于：
  1. 绑定M和P
  2. 从P维护的goroutine队列中获取goroutine，并执行任务调度

## 2. P的作用

- P维护了一个本地goroutine队列，能够实现近乎无锁化的goroutine调度
- P维护了一个mcache,在进行小内存分配时可以直接在mcache中获取，不需要加锁，实现了近乎无锁化的内存分配

P的本地队列是一个环形数组

```go
type p struct {
  runq [256]puintptr
  runqhead uint32
  runqtail uint32
}
```

Goroutine怎么加入到队列的？时，去全局队列获取任务。当本地队列和全局队列都没有任务时，会通过netpoll io库获取，如果都没有，则会去其他的p偷取一半的任务。

1. 通过go关键字创建goroutine
2. 等待事件阻塞的g,比如被channel事件的g事件触发时，会从等待链表中放回队列

## 3. Goroutine的生命周期

1. 创建，通过go关键字实现 _Gidle -> _Grunnable
2. 新创建新协程会优先加入到创建该协程的goroutine的本地队列（在调度时 _Grunnable -> _Grunning）
3. 当goroutine因channel/mutex阻塞时，会变更状态为 _Grunning -> _Gwaiting
4. 当goroutine唤醒时，会调用ready函数，将这个goroutine放到runnext位置，首先执行，并检查是否有空闲的p
5. 当本地p队列不足