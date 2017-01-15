准备开始
===============

这个项目尝试与[ReactiveX.io](http://reactivex.io/)保持一致。一般的跨平台文档和教程对于`RxSwift`来说也应该有效。

2. [Observables 又名 Sequences](#observables-aka-sequences)
1. [Disposing](#disposing)
1. [Implicit `Observable` guarantees](#implicit-observable-guarantees)
1. [Creating your first `Observable` (aka observable sequence)](#creating-your-own-observable-aka-observable-sequence)
1. [Creating an `Observable` that performs work](#creating-an-observable-that-performs-work)
1. [Sharing subscription and `shareReplay` operator](#sharing-subscription-and-sharereplay-operator)
1. [Operators](#operators)
1. [Playgrounds](#playgrounds)
1. [Custom operators](#custom-operators)
1. [Error handling](#error-handling)
1. [Debugging Compile Errors](#debugging-compile-errors)
1. [Debugging](#debugging)
1. [Debugging memory leaks](#debugging-memory-leaks)
1. [KVO](#kvo)
1. [UI layer tips](#ui-layer-tips)
1. [Making HTTP requests](#making-http-requests)
1. [RxDataSources](#rxdatasources)
1. [Driver](Units.md#driver-unit)
1. [Examples](Examples.md)

# Observables 又名 Sequences

## 基础
观察者模式(`Observable<Element>`序列)和普通序列(`SequenceType`）之间的相等性是理解Rx最重要的事情。

**每个 `Observable` 序列只是一个序列。一个 `Observable`相较 Swift 的`SequenceType` 的核心优势是它还能异步的接收元素。这是 RxSwift 的核心， 这里的文档是关于我们在那个想法上的展开。**

* `Observable`(`ObservableType`) 与 `SequenceType` 相等
* `ObservableType.subscribe` 方法与 `SequenceType.generate` 方法相等。
* Observer (callback) 需要被传进 `ObservableType.subscribe` 方法来接收序列元素，代替在返回的 generator 上调用 `next()` 方法.

序列是一个简单，常见的观念并且**易于视觉化**

人类是有巨大视区的生物。当我们能简单的视觉化一个观念，那么将非常容易解释它。

我们能从尝试通过序列模仿时间状态机内部每一个Rx操作符至高等级操作来提升认知负荷。

如果我们不用Rx而是异步模型系统，那么很可能意味着我们的代码将充满状态机和需要模拟而不是抽象的暂态

列表和序列大概是数学家和程序员首先学的概念之一

这里是一个数字序列：

```
--1--2--3--4--5--6--| // terminates normally 正常终止
```

另一个含有字符的序列：

```
--a--b--a--a--a---d---X // terminates with error 终止伴随错误
```

一些序列是有限的而另一些是无线的，比如一个按钮点击的事件序列：


```
---tap-tap-------tap--->
```

这被称为弹珠图。这里有更多的弹珠图[rxmarbles.com](http://rxmarbles.com)。

当我们需要指出序列语法作为一个常规表达式，他将会是这样：

**Next* (Error | Completed)?**

描述如下：

* **序列能有0或多个元素**
* **一旦一个 `Error` 或者 `Completed` 事件被接收， 那么序列将不能产生任何其他元素**

Rx中的序列被描述为一个推送接口（又名回调）。

```swift
enum Event<Element>  {
    case Next(Element)      // next element of a sequence 一个序列的next元素
    case Error(ErrorType)   // sequence failed with error 含有错误的序列
    case Completed          // sequence terminated successfully 成功终止的序列
}

class Observable<Element> {
    func subscribe(observer: Observer<Element>) -> Disposable
}

protocol ObserverType {
    func on(event: Event<Element>)
}
```

**当一个序列发送 `Completed` 或者 `Error` 事件，所有计算序列的内部资源将被释放**

**为了取消生产序列元素和立刻释放资源，可以在返回的订阅上调用 `dispose` 方法**

如果一个序列终止于有限次数，不调用 `dispose` 或者不使用 `addDisposableTo(disposeBag)` 将不会引起任何永久的资源泄露。无论如何，当这个序列完成时，这些资源都会被释放，或者通过处理完所有元素，或通过返回一个错误。

如果一个序列由于某些原因没有终止，资源将会被永久分配，除非 `dispose` 被手动调用，自动加入一个 `disposeBag`, `takeUntil` 或一些其他方法。

** 使用处置包或者 `takeUntil` 操作符是一个健壮的确认资源被释放的方法。我们建议在生产环境中使用他们，甚至序列将会在有限次数内终止。

如果你对为什么 `ErrorType` 不是泛型感动好奇，那么你可以在[这里](DesignRationale.md#why-error-type-isnt-generic)找到解释。

## 处置

一个被观察了的序列有一个额外的方法被终止。当我们完成一个序列然后想去释放所有已分配的资源，我们可以在一个订阅上调用 `dispose` 方法。


这是一个 `interval` 操作的例子：

```swift
let subscription = Observable<Int>.interval(0.3, scheduler: scheduler)
    .subscribe { event in
        print(event)
    }

NSThread.sleepForTimeInterval(2)

subscription.dispose()

```

This will print:

```
0
1
2
3
4
5
```

一般来讲你不想手动调用 `dispose`; 这只是一个教育例子。手动调用dispose一般来讲是一个坏做法。这里有更好的方法处理订阅。我们能使用 `DisposeBag`， `takeUntil` 操作符，或其他技巧。

那么这个代码在执行 `dispose` 之后能打印东西吗？答案是：看情况。

* 如果 `scheduler` 是一个 **系列调度** (比如，`MainScheduler`) 并且**相同的系列调度上**调用了 `dispose`，那么答案是**否**。 

* 否则答案是 **yes**

你能找出更多关于调度[here](Schedulers.md)

你可能有两个并行的进程。

* 一个在处理元素
* 另一个在处置订阅

“能在之后打印东西？”这个问题根本没有意义如果进程在两个不同的调度上。

一些更多的例子只是为了(`observeOn`的说明在[这](Schedulers.md))。

如果我们有如下：

```swift
let subscription = Observable<Int>.interval(0.3, scheduler: scheduler)
            .observeOn(MainScheduler.instance)
            .subscribe { event in
                print(event)
            }

// ....

subscription.dispose() // called from main thread

```

**在调用`dispose`返回之后，不会有任何打印。那是肯定的**

另外，如果有如下例子：

```swift
let subscription = Observable<Int>.interval(0.3, scheduler: scheduler)
            .observeOn(serialScheduler)
            .subscribe { event in
                print(event)
            }

// ...

subscription.dispose() // executing on same `serialScheduler`

```

**在调用`dispose`返回之后，不会有任何打印。这是肯定的**

### 处置包(Dispose Bags)

`Dispose bags`之于RX被用来返回类似ARC行为

当一个 `DisposeBag` 被销毁， 它会在每一个已加入的可处置对象中调用 `dispose` 方法。

因为他没有 `dispose` 方法，所以不允许在目标上显示调用 dispose 方法。如果需要马上清理，我们仅需创建一个新的dipose bag。

```swift
  self.disposeBag = DisposeBag()
```

这会清理旧的引用并且触发资源处理。

如果还是需要手动显示清理，使用 `CompositeDisposable`。 **它有被需要的行为但是一旦调用 `dispose` 方法，它会立刻处理任何新加入的可处置对象。

### Take until

还有一个销毁时自动处理订阅的方法，那就是使用 `takeUntil` 操作符。

```swift
sequence
    .takeUntil(self.rx_deallocated)
    .subscribe {
        print($0)
    }
```

## `Observable` 隐式惯例

这里还有一些另外的惯例，所有序列生产者(`Observable`们)都需要遵守。

无论生产者在哪个线程制造元素，如果他们生成一个元素并且发送给观察者(observer) `observer.on(.Next(nextElement))`, 他们不能发送下一个元素直到 `observer.on` 方法完成执行。

生产者也不能发送终止 `.Completed` 或者 `.Error`，如果 `.Next` 事件没有完成。

简而言之，考虑如下例子：

```swift
someObservable
  .subscribe { (e: Event<Element>) in
      print("Event processing started")
      // processing
      print("Event processing ended")
  }
```

打印永远如下：

```
Event processing started
Event processing ended
Event processing started
Event processing ended
Event processing started
Event processing ended
```

不可能出现如下打印：

```
Event processing started
Event processing started
Event processing ended
Event processing ended
```

## 创建你自己的 `Observable`

理解 `Observable` 还有一件很重要的事情。

**当一个 observable 被创建，他不简单的执行任何工作因为他已被创建了**

`Observable` 可以通过很多方法生成元素。他们其中一些会引起副作用，而另一些进入正存在运行的进程中，比如鼠标的点击事件，等等。

**但是如果你只是调用一个返回一个 `Observable` 的方法，生成序列不会被执行，并且不会有副作用。`Observable` 只是一个解释序列如果被生成和什么参数被使用于生成元素的定义。生成序列开始于 `subscribe` 方法被调用的时候。**

例如：让我们来说说一个有相似原型的方法：


```swift
func searchWikipedia(searchTerm: String) -> Observable<Results> {}
```

```swift
let searchForMe = searchWikipedia("me")

// no requests are performed, no work is being done, no URL requests were fired
// 请求没有被执行，不执行任何工作，URL请求没有被触发。

let cancel = searchForMe
  // sequence generation starts now, URL requests are fired
  // 生成序列现在开始，URL请求被触发
  .subscribeNext { results in
      print(results)
  }

```

这里有许多方法用于创建你自己的 `Observable` 序列。 最简单的方法大概就是使用 `create` 函数。

让我们创建一个函数，这个函数会创建一个序列并且返回一个可订阅的元素。这个函数被称为 'just'。

*这是实际的实现*

```swift
func myJust<E>(element: E) -> Observable<E> {
    return Observable.create { observer in
        observer.on(.Next(element))
        observer.on(.Completed)
        return NopDisposable.instance
    }
}

myJust(0)
    .subscribeNext { n in
      print(n)
    }
```

this will print:

```
0
```

不错。那什么是 `create` 函数？

他只是一个方便地使你使用Swift闭包来简单的实现 `subscribe` 方法。像 `subscribe` 方法需要一个参数，`observer`，并且返回 disposable。


实现序列这个方法实际上是同步的。他会生成元素并且在 `subscribe` 调用返回 disposable 订阅之前终止。

当生成同步的序列，通常返回的 disposable 是 `NopDisposable` 的单例实例。

现在来创建一个从数组中返回元素的 obserbable。

*这是实际的实现*

```swift
func myFrom<E>(sequence: [E]) -> Observable<E> {
    return Observable.create { observer in
        for element in sequence {
            observer.on(.Next(element))
        }

        observer.on(.Completed)
        return NopDisposable.instance
    }
}

let stringCounter = myFrom(["first", "second"])

print("Started ----")

// first time
stringCounter
    .subscribeNext { n in
        print(n)
    }

print("----")

// again
stringCounter
    .subscribeNext { n in
        print(n)
    }

print("Ended ----")
```

This will print:

```
Started ----
first
second
----
first
second
Ended ----
```

## 创建一个执行工作的 `Obserbable`

好的，现在来一些更有趣的事情。让我们创建一个 `interval` 操作符，他在之前的例子中有用到。

*这等价于实际的派遣队列调度实现*

```swift
func myInterval(interval: NSTimeInterval) -> Observable<Int> {
    return Observable.create { observer in
        print("Subscribed")
        let queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0)
        let timer = dispatch_source_create(DISPATCH_SOURCE_TYPE_TIMER, 0, 0, queue)

        var next = 0

        dispatch_source_set_timer(timer, 0, UInt64(interval * Double(NSEC_PER_SEC)), 0)
        let cancel = AnonymousDisposable {
            print("Disposed")
            dispatch_source_cancel(timer)
        }
        dispatch_source_set_event_handler(timer, {
            if cancel.disposed {
                return
            }
            observer.on(.Next(next))
            next += 1
        })
        dispatch_resume(timer)

        return cancel
    }
}
```

```swift
let counter = myInterval(0.1)

print("Started ----")

let subscription = counter
    .subscribeNext { n in
       print(n)
    }

NSThread.sleepForTimeInterval(0.5)

subscription.dispose()

print("Ended ----")
```

This will print
```
Started ----
Subscribed
0
1
2
3
4
Disposed
Ended ----
```

假设你这么写

```swift
let counter = myInterval(0.1)

print("Started ----")

let subscription1 = counter
    .subscribeNext { n in
       print("First \(n)")
    }
let subscription2 = counter
    .subscribeNext { n in
       print("Second \(n)")
    }

NSThread.sleepForTimeInterval(0.5)

subscription1.dispose()

NSThread.sleepForTimeInterval(0.5)

subscription2.dispose()

print("Ended ----")
```

this would print:

```
Started ----
Subscribed
Subscribed
First 0
Second 0
First 1
Second 1
First 2
Second 2
First 3
Second 3
First 4
Second 4
Disposed
Second 5
Second 6
Second 7
Second 8
Second 9
Disposed
Ended ----
```

**每一个订阅了的订阅者通常生成他自己分片元素。操作符默认是无状态的。无状态的操作符远远多于有状态的。**

## 分享订阅和 `shareReplay` 操作符

但是如果你想要多个观察者从一个订阅中分享事件？

这有两件事需要被定义

* 如何处理那些新的订阅者被观察之前已经收到的元素。（只重放最近的，重放所有，重放最近n个）
* 如果决定什么时候出发分享的订阅（引用计数，手动或者一些其他算法）

通常的选择是一个`replay(1).refCount()` 又名 `shareReplay()`的组合。

```swift
let counter = myInterval(0.1)
    .shareReplay(1)

print("Started ----")

let subscription1 = counter
    .subscribeNext { n in
       print("First \(n)")
    }
let subscription2 = counter
    .subscribeNext { n in
       print("Second \(n)")
    }

NSThread.sleepForTimeInterval(0.5)

subscription1.dispose()

NSThread.sleepForTimeInterval(0.5)

subscription2.dispose()

print("Ended ----")
```

this will print

```
Started ----
Subscribed
First 0
Second 0
First 1
Second 1
First 2
Second 2
First 3
Second 3
First 4
Second 4
First 5
Second 5
Second 6
Second 7
Second 8
Second 9
Disposed
Ended ----
```

注意现在只有一个 `Subscribed` 和 `Disposed` 事件。

URL 的 observables 行为是一样的。

这是HTTP请求如果用Rx封装的。这和 `interval` 操作符的模式非常相似。

```swift
extension NSURLSession {
    public func rx_response(request: NSURLRequest) -> Observable<(NSData, NSURLResponse)> {
        return Observable.create { observer in
            let task = self.dataTaskWithRequest(request) { (data, response, error) in
                guard let response = response, data = data else {
                    observer.on(.Error(error ?? RxCocoaURLError.Unknown))
                    return
                }

                guard let httpResponse = response as? NSHTTPURLResponse else {
                    observer.on(.Error(RxCocoaURLError.NonHTTPResponse(response: response)))
                    return
                }

                observer.on(.Next(data, httpResponse))
                observer.on(.Completed)
            }

            task.resume()

            return AnonymousDisposable {
                task.cancel()
            }
        }
    }
}
```

## 操作符

RxSwift中有许多实现了的操作符。完整列表在[这](API.md)

所有操作的 Marble 图可以在[ReactiveX.io](http://reactivex.io/)上找到。

大多数操作符的演示在[Playgrounds](../Rx.playground)。

使用 playgrounds 请打开 `Rx.xcworkspace`， 构建 `RxSwift-OSX` scheme 然后在 `Rx.xcworkspace` 视图树种打开 playgrounds。

如果你需要一个操作符，但是不知道如何找到他，这里有一个[操作符大全](http://reactivex.io/documentation/operators.html#tree)

[RxSwift支持的操作符](API.md#rxswift-supported-operators)也是按照他们的功能被分组，那也是有些帮助。

### 自定义操作符

这有两种创建自定义操作符的方法。

#### 简单方法

所有内部代码使用高度优化了的版本的操作符，所以他们不是最好的教程原料。这也是为什么高度鼓励使用标准操作符的原因。

幸运的是这有一个更简单的方法创建操作符。创建操作符其实就是创建 observables， 并且之前的章节已经描述了如何操作了。

让我们看看如何实现一个未被优化的 map 操作。

```swift
extension ObservableType {
    func myMap<R>(transform: E -> R) -> Observable<R> {
        return Observable.create { observer in
            let subscription = self.subscribe { e in
                    switch e {
                    case .Next(let value):
                        let result = transform(value)
                        observer.on(.Next(result))
                    case .Error(let error):
                        observer.on(.Error(error))
                    case .Completed:
                        observer.on(.Completed)
                    }
                }

            return subscription
        }
    }
}
```

所以现在你能看到自己的 map :

```swift
let subscription = myInterval(0.1)
    .myMap { e in
        return "This is simply \(e)"
    }
    .subscribeNext { n in
        print(n)
    }
```

并且会打印

```
Subscribed
This is simply 0
This is simply 1
This is simply 2
This is simply 3
This is simply 4
This is simply 5
This is simply 6
This is simply 7
This is simply 8
...
```

### 权宜之计

所以如果用自定义操作符解决问题太困难？你可以退出 Rx monad，在 imperative world 执行行动，然后将结果装换成 Rx 中的 `Subject`。

这是不应该被经常被实践的，并且是坏代码的例子，但是你可以做：

```swift
  let magicBeings: Observable<MagicBeing> = summonFromMiddleEarth()

  magicBeings
    .subscribeNext { being in     // exit the Rx monad  
        self.doSomeStateMagic(being)
    }
    .addDisposableTo(disposeBag)

  //
  //  Mess
  //
  let kitten = globalParty(   // calculate something in messy world
    being,
    UIApplication.delegate.dataSomething.attendees
  )
  kittens.on(.Next(kitten))   // send result back to rx
  //
  // Another mess
  //

  let kittens = Variable(firstKitten) // again back in Rx monad

  kittens.asObservable()
    .map { kitten in
      return kitten.purr()
    }
    // ....
```

每次你这么做，很有可能会有人这么写：

```swift
  kittens
    .subscribeNext { kitten in
      // so something with kitten
    }
    .addDisposableTo(disposeBag)
```

所以请不要尝试这么做。

## Playgrounds

如果你不确定一些操作符是如何工作的，[playgrounds](../Rx.playground) 包含
几乎所有操作符已经准备好的小例子来举例说明他们的行为。

**要使用 playgrounds 请打开 Rx.xcworkspace，构建 RxSwift-OSX scheme 并且打开 Rx.xcworkspace 树视图中的 playgrounds。**

**想要查看 playgrounds 例子的结果，请打开 `Assistant Editor`。你可以通过点击 `View > Assistant Editor > Show Assistant Editor` 打开 `Assistant Editor`**

## 错误处理

有两个错误机制。

### observables的异步错误处理机制

错误处理是非常直截了当的。如果一个序列终止于错误，那么所有想依赖的序列都会终止于错误。这是普通的短回路逻辑。

你可以使用 `catch` 操作符接收失败的 observable。

如果有错误的序列还能 `retry` 操作符重试。

## 调试编译错误

当编写大量RxSwift/RxCocoa的代码时，你可能重度依靠编译器去推测`Observable`s的类型。这也是为什么Swift很棒的原因之一，但是这有时候也会令人沮丧。

```swift
images = word
    .filter { $0.containsString("important") }
    .flatMap { word in
        return self.api.loadFlickrFeed("karate")
            .catchError { error in
                return just(JSON(1))
            }
      }
```

如果编译器报告说这个表达式什么地方存在一个错误，我会首先建议声明返回类型。

```swift
images = word
    .filter { s -> Bool in s.containsString("important") }
    .flatMap { word -> Observable<JSON> in
        return self.api.loadFlickrFeed("karate")
            .catchError { error -> Observable<JSON> in
                return just(JSON(1))
            }
      }
```

如果不管用，那么你可以继续添加更多的类型声明，知道你定位了错误。

```swift
images = word
    .filter { (s: String) -> Bool in s.containsString("important") }
    .flatMap { (word: String) -> Observable<JSON> in
        return self.api.loadFlickrFeed("karate")
            .catchError { (error: NSError) -> Observable<JSON> in
                return just(JSON(1))
            }
      }
```

**我建议首先声明闭包的返回类型和参数。

通常在你修正了错误之后，你也可以移除类型声明，清理你的代码。

## 调试

单独使用调试器非常有用，但是常常使用 `debug` 操作符会更有效。 `debug` 操作符将所有事件打印到标准输出并且还可以给那些事件增加标签。

`debug` acts like a probe. Here is an example of using it:
`debug` 扮演一个类似探针。下面是使用他的例子：

```swift
let subscription = myInterval(0.1)
    .debug("my probe")
    .map { e in
        return "This is simply \(e)"
    }
    .subscribeNext { n in
        print(n)
    }

NSThread.sleepForTimeInterval(0.5)

subscription.dispose()
```

将会打印

```
[my probe] subscribed
Subscribed
[my probe] -> Event Next(Box(0))
This is simply 0
[my probe] -> Event Next(Box(1))
This is simply 1
[my probe] -> Event Next(Box(2))
This is simply 2
[my probe] -> Event Next(Box(3))
This is simply 3
[my probe] -> Event Next(Box(4))
This is simply 4
[my probe] dispose
Disposed
```

你可以简单地创建自己版本的 `debug` 操作符。

```swift
extension ObservableType {
    public func myDebug(identifier: String) -> Observable<Self.E> {
        return Observable.create { observer in
            print("subscribed \(identifier)")
            let subscription = self.subscribe { e in
                print("event \(identifier)  \(e)")
                switch e {
                case .Next(let value):
                    observer.on(.Next(value))

                case .Error(let error):
                    observer.on(.Error(error))

                case .Completed:
                    observer.on(.Completed)
                }
            }
            return AnonymousDisposable {
                   print("disposing \(identifier)")
                   subscription.dispose()
            }
        }
    }
 }
```

## 调试内存泄露

调式模式中Rx使用一个全局变量 `resourceCount` 跟踪所有已分配的资源。

如果你想要一些资源泄露的探测逻辑，最简单的方法就是周期性地打印出 `RxSwift.resourceCount` 的值。

```swift
    /* add somewhere in
    func application(application: UIApplication, didFinishLaunchingWithOptions launchOptions: [NSObject: AnyObject]?) -> Bool
    */
    _ = Observable<Int>.interval(1, scheduler: MainScheduler.instance)
        .subscribeNext { _ in
        print("Resource count \(RxSwift.resourceCount)")
    }
```

最有效的测试内存泄露的方法是：
* 导航到你的屏幕并且使用它
* 导航回去
* 观察初始资源数（resource count）
* 第二次导航回你的屏幕并且使用它
* 导航回去
* 观察最终的资源数（resource count）

假如初始资源数不同于最终资源数，那么就有可能是内存泄露。

为什么建议切换两次导航，是因为第一次导航会强制加载lazy资源。

## 变量（Variables）

`Variable`s 代表一些 observable 的状态。`Variable` 必须包含一个值，因为初始化需要一个初始值。

Variable 封装了一个 [`Subject`](http://reactivex.io/documentation/subject.html)。 更明确的说他是一个 `BehaviorSubject`。不同于 `BehaviorSubject`，它只暴露 `value` 接口，所以 variable 永远不会终止或者失败。

在订阅中它还会立刻广播它当前的值。

在 variable 被释放之后，他会从 `.asObservable()` 的返回完成 observable 序列。

```swift
let variable = Variable(0)

print("Before first subscription ---")

_ = variable.asObservable()
    .subscribe(onNext: { n in
        print("First \(n)")
    }, onCompleted: {
        print("Completed 1")
    })

print("Before send 1")

variable.value = 1

print("Before second subscription ---")

_ = variable.asObservable()
    .subscribe(onNext: { n in
        print("Second \(n)")
    }, onCompleted: {
        print("Completed 2")
    })

variable.value = 2

print("End ---")
```

将打印

```
Before first subscription ---
First 0
Before send 1
First 1
Before second subscription ---
Second 1
First 2
Second 2
End ---
Completed 1
Completed 2
```

## KVO

KVO是一个 Objective-C 机制。这意味着它不是构建在类型安全上的。这个项目尝试去解决一些这类问题。

下面是两种这个库支持的KVO构建方法：

```swift
// KVO
extension NSObject {
    public func rx_observe<E>(type: E.Type, _ keyPath: String, options: NSKeyValueObservingOptions, retainSelf: Bool = true) -> Observable<E?> {}
}

#if !DISABLE_SWIZZLING
// KVO
extension NSObject {
    public func rx_observeWeakly<E>(type: E.Type, _ keyPath: String, options: NSKeyValueObservingOptions) -> Observable<E?> {}
}
#endif
```

如何观察 `UIView` 的frame的例子。

**警告：UIKit不遵从KVO，但是它可以工作。**

```swift
view
  .rx_observe(CGRect.self, "frame")
  .subscribeNext { frame in
    ...
  }
```

or

```swift
view
  .rx_observeWeakly(CGRect.self, "frame")
  .subscribeNext { frame in
    ...
  }
```

### `rx_observe`

`rx_observe` 是更高效的，因为他只是一个KVO机制的简单封装，但是也更局限了他的使用场景。

* 他可以被用来观察所有权图(`retainSelf = false`)中从 `self` 或者其祖先开始的路径
* 他可以被用来观察所有权图(`retainSelf = true `)中从其子类开始的路径
* 路径中不得不只能包含 `strong` 属性，否则你的系统就有崩溃的风险，因为没有在释放前注册KVO观察者。

E.g.

```swift
self.rx_observe(CGRect.self, "view.frame", retainSelf: false)
```

### `rx_observeWeakly`

`rx_observeWeakly` 比 `rx_observe` 有一点慢，因为如果是弱引用它不得不处理对象的释放。

它可以被用在所有 `rx_observe` 可以用的地方并且除此之外

* 因为它不会持有被观察目标的引用，所以它能被用来观察任意产权关系不明的对象图

E.g.

```swift
someSuspiciousViewController.rx_observeWeakly(Bool.self, "behavingOk")
```

### Observing structs

KVO 是一个 Objective-C 机制，所以它中毒依赖 `NSValue`。

**RxCocoa 支持 `CGRect`，`CGSize` 和 `CGPoint` 结构的KVO。**

当观察其他结构，需要手动从 `NSValue` 提取那些结构。

[这里](../RxCocoa/Common/KVORepresentable+CoreGraphics.swift)是如何通过实现 `KVORepresentable` 协议，为其他结构扩展KVO观察机制和 `rx_observe*` 方法的例子。

## UI层小贴士

当绑定 UIKit 控制时，有一件很明确的事情是，你的 `Observable`s 需要在UI层令人满意。

### 线程

`Observable`s 需要在 `MainScheduler`(UIThread) 上发送值。 那只是一个普通的 UIKit/Cocoa 的需要。

通常来说，你的 APIs 在 `MainScheduler` 上返回结果是一个不错的主意。假如你想尝试从后台线程中绑定东西到UI，在 **Debug** 构建RxCocoa时，通常会抛出一个异常来提醒你这点。

你需要增加 `observeOn(MainScheduler.instance)` 来修正这点。

**NSURLSession扩展默认不在 `MainScheduler` 上返回结果。**

### Errors

你不能绑定失败结果到 UIKit 控制，因为那是一个未定义的行为。

如果你不知道 `Observable` 是否会失败，你可以使用 `catchErrorJustReturn(valueThatIsReturnedWhenErrorHappens)` 来确保他不会失败，**但是当一个错误发生之后，之后的序列还是会完成**

如果想要之后的序列继续产生元素，那就需要 `retry` 操作符。

### 分享订阅

你通常想要在UI层分享订阅。你不需要去调用不同的HTTP请求来绑定同样的数据到多个UI元素上去。

你想要下面这些：

```swift
let searchResults = searchText
    .throttle(0.3, $.mainScheduler)
    .distinctUntilChanged
    .flatMapLatest { query in
        API.getSearchResults(query)
            .retry(3)
            .startWith([]) // clears results on new search term
            .catchErrorJustReturn([])
    }
    .shareReplay(1)              // <- notice the `shareReplay` operator
```

通常你需要的是在计算后分享搜索结果。那就是 `shareReplay` 的意义。

**对于UI层通常来说在转换链的最后添加 `shareReplay` 是一个好的规则，因为你实在想要分享计算后的结果。当时绑定 `searchResults` 到多个UI元素时，你不必出发单独的HTTP连接。

**另外看一下 `Driver` 单元。它被设计用来直白的封装那些 `shareReply` 调用，确保元素被观察在UI主线程并且不会有错误被绑定到UI**

## 制作 HTTP 请求

制作 http 请求是大家第一件要尝试的事情之一。

你首先需要构建 `NSURLRequest` 对象。

请求决定了它是一个 GET 请求或者是 POST 请求，请求体是什么，查询参数是什么 ...

下面是你如何能创建一个简单的 GET 请求

```swift
let request = NSURLRequest(URL: NSURL(string: "http://en.wikipedia.org/w/api.php?action=parse&page=Pizza&format=json")!)
```

如果你仅仅想用其他 observables 执行请求外部成分，下面就是你需要做的。

```swift
let responseJSON = NSURLSession.sharedSession().rx_JSON(request)

// no requests will be performed up to this point
// `responseJSON` is just a description how to fetch the response

let cancelRequest = responseJSON
    // this will fire the request
    .subscribeNext { json in
        print(json)
    }

NSThread.sleepForTimeInterval(3)

// if you want to cancel request after 3 seconds have passed just call
cancelRequest.dispose()

```

**NSURLSession 扩展默认不在 `MainScheduler` 上返回结果。**

假如你想要一个更低级的响应访问，你可以使用：

```swift
NSURLSession.sharedSession().rx_response(myNSURLRequest)
    .debug("my request") // this will print out information to console
    .flatMap { (data: NSData!, response: NSURLResponse!) -> Observable<String> in
        if let response = response as? NSHTTPURLResponse {
            if 200 ..< 300 ~= response.statusCode {
                return just(transform(data))
            }
            else {
                return Observable.error(yourNSError)
            }
        }
        else {
            rxFatalError("response = nil")
            return Observable.error(yourNSError)
        }
    }
    .subscribe { event in
        print(event) // if error happened, this will also print out error to console
    }
```

### HTTP 通信日志

在调试模式中，RxCocoa 默认将会记录所有的 HTTP 请求日志到控制台上。如果你要改变这个行为，请设置 `Logging.URLRequests` 筛选器。

```swift
// read your own configuration
public struct Logging {
    public typealias LogURLRequest = (NSURLRequest) -> Bool

    public static var URLRequests: LogURLRequest =  { _ in
    #if DEBUG
        return true
    #else
        return false
    #endif
    }
}
```

## RxDataSources

... 是一个实现 `UITableView`s 和 `UICollectionView` 全部功能的交互数据源的类集合。


RxDataSources 开源在[这](https://github.com/RxSwiftCommunity/RxDataSources)。

如何使用它们的全部功能示范在 [RxExample](../RxExample) 项目中。
