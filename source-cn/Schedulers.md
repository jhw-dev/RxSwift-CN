调度者（Schedulers）
==========

1. [Serial vs Concurrent Schedulers](#serial-vs-concurrent-schedulers)
1. [Custom schedulers](#custom-schedulers)
1. [Builtin schedulers](#builtin-schedulers)

Schedulers abstract away mechanism for performing work.
调度者抽象了执行工作的机制。

Different mechanisms for performing work include, current thread, dispatch queues, operation queues, new threads, thread pools, run loops ...
执行工作不同的机制包括同线程，派遣队列，操作队列，创建线程，线程池，运行循环等

There are two main operators that work with schedulers. `observeOn` and `subscribeOn`.
调度者有两个主要的操作。`observeOn` 和 `subscribeOn`。

If you want to perform work on different scheduler just use `observeOn(scheduler)` operator.
如果你想要在不同的调度者上执行工作，只需要调用 `observeOn(scheduler)` 操作符。

You would usually use `observeOn` a lot more often then `subscribeOn`.
你将经常使用 `observeOn` 大大多于使用 `subscribeOn`。

In case `observeOn` isn't explicitly specified, work will be performed on which ever thread/scheduler elements are generated.
假如 `observeOn` 没有显示指定， 那么任务将被执行在生成元素的调度器或线程上。

Example of using `observeOn` operator
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

If you want to start sequence generation (`subscribe` method) and call dispose on a specific scheduler, use `subscribeOn(scheduler)`.
如果你需要开始序列生成（ `subscribe` 方法）并且调用处置方法在特定的调度器上，使用 `subscribeOn(scheduler)`。

In case `subscribeOn` isn't explicitly specified, `subscribe` method will be called on the same thread/scheduler that `subscribeNext` or `subscribe` is called.
如果 `subscribeOn` 没有显示指定，`subscribe` 方法会被调用在执行 `subscribeNext` or `subscribe` 的同一个线程或者调度器上。

In case `subscribeOn` isn't explicitly specified, `dispose` method will be called on the same thread/scheduler that initiated disposing.
如果 `subscribeOn` 没有显示指定，`dispose ` 方法会被调用在执行初始化处置（disposing）的线程和调度器上。

In short, if no explicit scheduler is chosen, those methods will be called on current thread/scheduler.
简而言之，如果没有显示的调度器被选择，那些方法会被调用在相同的线程或调度器上。

# Serial vs Concurrent Schedulers
# 串行和并发调度器

Since schedulers can really be anything, and all operators that transform sequences need to preserve additional [implicit guarantees](GettingStarted.md#implicit-observable-guarantees), it is important what kind of schedulers are you creating.
由于调度器真的能够成为任何东西，并且所有改变序列的操作符需要保留格外的[隐性惯例](GettingStarted.md#implicit-observable-guarantees), 你创建什么类型的调度器是非常重要的。

In case scheduler is concurrent, Rx's `observeOn` and `subscribeOn` operators will make sure everything works perfect.
假如调度器是并发的，Rx的 `observeOn` 和 `subscribeOn` 操作符将会确保所有工作正常。

If you use some scheduler that for which Rx can prove that it's serial, it will able to perform additional optimizations.
如果你使用一些Rx能保证串行的调度器，这能执行额外的优化。

So far it only performing those optimizations for dispatch queue schedulers.
迄今为止只对派遣队列调度器实行了那些优化。

In case of serial dispatch queue schedulers `observeOn` is optimized to just a simple `dispatch_async` call.
串行派遣队列调度器仅仅使用 `dispatch_async` 调用来优化 `observeOn`。

# Custom schedulers
# 自定义调度器

Besides current schedulers, you can write your own schedulers.
除了当前的调度器意外，你可以写属于自己的调度器。

If you just want to describe who needs to perform work immediately, you can create your own scheduler by implementing `ImmediateScheduler` protocol.
如果你想要描述立刻执行工作的调度器，你可以通过实现 `ImmediateScheduler` 协议创建属于自己的调度器。

```swift
public protocol ImmediateScheduler {
    func schedule<StateType>(state: StateType, action: (/*ImmediateScheduler,*/ StateType) -> RxResult<Disposable>) -> RxResult<Disposable>
}
```

If you want to create new scheduler that supports time based operations, then you'll need to implement.
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

In case scheduler only has periodic scheduling capabilities, you can inform Rx by implementing `PeriodicScheduler` protocol
假如调度器只有周期性调度的能力，你可以通过实现 Rx 的 `PeriodicScheduler` 协议。

```swift
public protocol PeriodicScheduler : Scheduler {
    func schedulePeriodic<StateType>(state: StateType, startAfter: TimeInterval, period: TimeInterval, action: (StateType) -> StateType) -> RxResult<Disposable>
}
```

In case scheduler doesn't support `PeriodicScheduling` capabilities, Rx will emulate periodic scheduling transparently.
假如调度器不支持 `PeriodicScheduling` 的能力，Rx 将会不给察觉的模拟周期性调度。

# Builtin schedulers
# 内置调度器

Rx can use all types of schedulers, but it can also perform some additional optimizations if it has proof that scheduler is serial.
Rx 可以使用所有类型的调度器，但是它还能执行一些额外的优化，如果他能证明调度器是串行的。

These are currently supported schedulers
这些是当前支持的调度器

## CurrentThreadScheduler (Serial scheduler)
## 当前线程调度器（CurrentThreadScheduler）串行调度器

Schedules units of work on the current thread.
This is the default scheduler for operators that generate elements.
工作的调度器单位在当前线程上。
这是生成元素操作的默认调度器。

This scheduler is also sometimes called `trampoline scheduler`.
这个调度器有时候也被称为 `蹦床调度器`。

If `CurrentThreadScheduler.instance.schedule(state) { }` is called for first time on some thread, scheduled action will be executed immediately and hidden queue will be created where all recursively scheduled actions will be temporarily enqueued.
如果 `CurrentThreadScheduler.instance.schedule(state) { }` 首次被调用在同一个线程上，调度的行为会被立刻执行，并且隐藏的队列会被创建，所有递归调度操作将暂时入队。

If some parent frame on call stack is already running `CurrentThreadScheduler.instance.schedule(state) { }`, scheduled action will be enqueued and executed when currently running action and all previously enqueued actions have finished executing.
如果一些父结构在调用堆上已经正在执行 `CurrentThreadScheduler.instance.schedule(state) { }`，调度行为将会被入队并且执行，在当前运行行为的时候，并且所有先前的入队行为已经完成执行。

## MainScheduler (Serial scheduler)
## 主调度器（MainScheduler）（串行调度器）

Abstracts work that needs to be performed on `MainThread`. In case `schedule` methods are called from main thread, it will perform action immediately without scheduling.
抽象工作需要被执行在 `MainThread`。假如 `schedule` 方法被调用在主线程，他就会直接执行动作不再调度。

This scheduler is usually used to perform UI work.
这个调度器经常被用在执行 UI 任务。

## SerialDispatchQueueScheduler (Serial scheduler)
## 串行派遣队列调度器（SerialDispatchQueueScheduler）（串行调度）

Abstracts the work that needs to be performed on a specific `dispatch_queue_t`. It will make sure that even if concurrent dispatch queue is passed, it's transformed into a serial one.
抽象工作需要在指定的 `dispatch_queue_t` 上被执行。他会确认即使派遣队列被同时传递，他也将变换成一个串行（线性）的。

Serial schedulers enable certain optimizations for `observeOn`.
串行调度器允许为了 `observeOn` 确定的优化。

Main scheduler is an instance of `SerialDispatchQueueScheduler`.
主调度器是一个 `SerialDispatchQueueScheduler` 的实例。

## ConcurrentDispatchQueueScheduler (Concurrent scheduler)
## 并行调度器队列调度器（ConcurrentDispatchQueueScheduler）（并发调度器）

Abstracts the work that needs to be performed on a specific `dispatch_queue_t`. You can also pass a serial dispatch queue, it shouldn't cause any problems.
抽象工作需要被执行在指定的 `dispatch_queue_t` 上。你还可以传递一个串行派遣队列，他不应该引起任务问题。

This scheduler is suitable when some work needs to be performed in background.
这个调度器适合在当一些任务需要被执行在后台。

## OperationQueueScheduler (Concurrent scheduler)
## 操作队列调度器（OperationQueueScheduler）（并发调度器）

Abstracts the work that needs to be performed on a specific `NSOperationQueue`.
抽象工作需要被执行在指定的 `NSOperationQueue` 上。

This scheduler is suitable for cases when there is some bigger chunk of work that needs to be performed in background and you want to fine tune concurrent processing using `maxConcurrentOperationCount`.
这个调度器适合在有一些很大的块工作需要被执行在后台，并且你需要使用 `maxConcurrentOperationCount` 微调并发处理。