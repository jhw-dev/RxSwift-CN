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
如果你使用一些调度器

So far it only performing those optimizations for dispatch queue schedulers.

In case of serial dispatch queue schedulers `observeOn` is optimized to just a simple `dispatch_async` call.

# Custom schedulers

Besides current schedulers, you can write your own schedulers.

If you just want to describe who needs to perform work immediately, you can create your own scheduler by implementing `ImmediateScheduler` protocol.

```swift
public protocol ImmediateScheduler {
    func schedule<StateType>(state: StateType, action: (/*ImmediateScheduler,*/ StateType) -> RxResult<Disposable>) -> RxResult<Disposable>
}
```

If you want to create new scheduler that supports time based operations, then you'll need to implement.

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

```swift
public protocol PeriodicScheduler : Scheduler {
    func schedulePeriodic<StateType>(state: StateType, startAfter: TimeInterval, period: TimeInterval, action: (StateType) -> StateType) -> RxResult<Disposable>
}
```

In case scheduler doesn't support `PeriodicScheduling` capabilities, Rx will emulate periodic scheduling transparently.

# Builtin schedulers

Rx can use all types of schedulers, but it can also perform some additional optimizations if it has proof that scheduler is serial.

These are currently supported schedulers

## CurrentThreadScheduler (Serial scheduler)

Schedules units of work on the current thread.
This is the default scheduler for operators that generate elements.

This scheduler is also sometimes called `trampoline scheduler`.

If `CurrentThreadScheduler.instance.schedule(state) { }` is called for first time on some thread, scheduled action will be executed immediately and hidden queue will be created where all recursively scheduled actions will be temporarily enqueued.

If some parent frame on call stack is already running `CurrentThreadScheduler.instance.schedule(state) { }`, scheduled action will be enqueued and executed when currently running action and all previously enqueued actions have finished executing.

## MainScheduler (Serial scheduler)

Abstracts work that needs to be performed on `MainThread`. In case `schedule` methods are called from main thread, it will perform action immediately without scheduling.

This scheduler is usually used to perform UI work.

## SerialDispatchQueueScheduler (Serial scheduler)

Abstracts the work that needs to be performed on a specific `dispatch_queue_t`. It will make sure that even if concurrent dispatch queue is passed, it's transformed into a serial one.

Serial schedulers enable certain optimizations for `observeOn`.

Main scheduler is an instance of `SerialDispatchQueueScheduler`.

## ConcurrentDispatchQueueScheduler (Concurrent scheduler)

Abstracts the work that needs to be performed on a specific `dispatch_queue_t`. You can also pass a serial dispatch queue, it shouldn't cause any problems.

This scheduler is suitable when some work needs to be performed in background.

## OperationQueueScheduler (Concurrent scheduler)

Abstracts the work that needs to be performed on a specific `NSOperationQueue`.

This scheduler is suitable for cases when there is some bigger chunk of work that needs to be performed in background and you want to fine tune concurrent processing using `maxConcurrentOperationCount`.
