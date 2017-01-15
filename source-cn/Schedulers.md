调度者（Schedulers）
==========

1. [Serial vs Concurrent Schedulers](#serial-vs-concurrent-schedulers)
1. [Custom schedulers](#custom-schedulers)
1. [Builtin schedulers](#builtin-schedulers)

调度者抽象了执行工作的机制。

执行工作不同的机制包括同线程，派遣队列，操作队列，创建线程，线程池，运行循环等

调度者有两个主要的操作。`observeOn` 和 `subscribeOn`。

如果你想要在不同的调度者上执行工作，只需要调用 `observeOn(scheduler)` 操作符。

你将经常使用 `observeOn` 大大多于使用 `subscribeOn`。

假如 `observeOn` 没有显示指定， 那么任务将被执行在生成元素的调度器或线程上。

使用 `observeOn` 操作符的例子

```
sequence1
  .observeOn(backgroundScheduler)
  .map { n in
      print("This is performed on background scheduler")
  }
  .observeOn(MainScheduler.instance)
  .map { n in
      print("This is performed on main scheduler")
  }
```

如果你需要开始序列生成（ `subscribe` 方法）并且调用处置方法在特定的调度器上，使用 `subscribeOn(scheduler)`。

如果 `subscribeOn` 没有显示指定，`subscribe` 方法会被调用在执行 `subscribeNext` or `subscribe` 的同一个线程或者调度器上。

如果 `subscribeOn` 没有显示指定，`dispose ` 方法会被调用在执行初始化处置（disposing）的线程和调度器上。

简而言之，如果没有显示的调度器被选择，那些方法会被调用在相同的线程或调度器上。

# 串行和并发调度器

由于调度器真的能够成为任何东西，并且所有改变序列的操作符需要保留格外的[隐性惯例](GettingStarted.md#implicit-observable-guarantees), 你创建什么类型的调度器是非常重要的。

假如调度器是并发的，Rx的 `observeOn` 和 `subscribeOn` 操作符将会确保所有工作正常。

如果你使用一些Rx能保证串行的调度器，这能执行额外的优化。

迄今为止只对派遣队列调度器实行了那些优化。

串行派遣队列调度器仅仅使用 `dispatch_async` 调用来优化 `observeOn`。

# 自定义调度器

除了当前的调度器意外，你可以写属于自己的调度器。

如果你想要描述立刻执行工作的调度器，你可以通过实现 `ImmediateScheduler` 协议创建属于自己的调度器。

```swift
public protocol ImmediateScheduler {
    func schedule<StateType>(state: StateType, action: (/*ImmediateScheduler,*/ StateType) -> RxResult<Disposable>) -> RxResult<Disposable>
}
```

如果你想创建新的支持基于时间操作的调度器，然后你需要实现。

```swift
public protocol Scheduler: ImmediateScheduler {
    associatedtype TimeInterval
    associatedtype Time

    var now : Time {
        get
    }

    func scheduleRelative<StateType>(state: StateType, dueTime: TimeInterval, action: (StateType) -> RxResult<Disposable>) -> RxResult<Disposable>
}
```

假如调度器只有周期性调度的能力，你可以通过实现 Rx 的 `PeriodicScheduler` 协议。

```swift
public protocol PeriodicScheduler : Scheduler {
    func schedulePeriodic<StateType>(state: StateType, startAfter: TimeInterval, period: TimeInterval, action: (StateType) -> StateType) -> RxResult<Disposable>
}
```

假如调度器不支持 `PeriodicScheduling` 的能力，Rx 将会不给察觉的模拟周期性调度。

# 内置调度器

Rx 可以使用所有类型的调度器，但是它还能执行一些额外的优化，如果他能证明调度器是串行的。

这些是当前支持的调度器

## 当前线程调度器（CurrentThreadScheduler）串行调度器

工作的调度器单位在当前线程上。
这是生成元素操作的默认调度器。

这个调度器有时候也被称为 `蹦床调度器`。

如果 `CurrentThreadScheduler.instance.schedule(state) { }` 首次被调用在同一个线程上，调度的行为会被立刻执行，并且隐藏的队列会被创建，所有递归调度操作将暂时入队。

如果一些父结构在调用堆上已经正在执行 `CurrentThreadScheduler.instance.schedule(state) { }`，调度行为将会被入队并且执行，在当前运行行为的时候，并且所有先前的入队行为已经完成执行。

## 主调度器（MainScheduler）（串行调度器）

抽象工作需要被执行在 `MainThread`。假如 `schedule` 方法被调用在主线程，他就会直接执行动作不再调度。

这个调度器经常被用在执行 UI 任务。

## 串行派遣队列调度器（SerialDispatchQueueScheduler）（串行调度）

抽象工作需要在指定的 `dispatch_queue_t` 上被执行。他会确认即使派遣队列被同时传递，他也将变换成一个串行（线性）的。

串行调度器允许为了 `observeOn` 确定的优化。

主调度器是一个 `SerialDispatchQueueScheduler` 的实例。

## 并行调度器队列调度器（ConcurrentDispatchQueueScheduler）（并发调度器）

抽象工作需要被执行在指定的 `dispatch_queue_t` 上。你还可以传递一个串行派遣队列，他不应该引起任务问题。

这个调度器适合在当一些任务需要被执行在后台。

## 操作队列调度器（OperationQueueScheduler）（并发调度器）

抽象工作需要被执行在指定的 `NSOperationQueue` 上。

这个调度器适合在有一些很大的块工作需要被执行在后台，并且你需要使用 `maxConcurrentOperationCount` 微调并发处理。
