Units
=====

这个文档将尝试描述 units 是什么，为什么他们是有用的理念，并且如果使用和创建他们。

* [Why](#why)
* [How they work](#how-they-work)
* [Why they are named Units](#why-they-are-named-units)
* [RxCocoa units](#rxcocoa-units)
* [Driver unit](#driver-unit)
    * [Why it's named Driver](#why-its-named-driver)
    * [Practical usage example](#practical-usage-example)

## Why

Swift 拥有一个强大的类型系统，这能用来提高正确性和应用的稳定性，并且使使用 Rx 成为更直观和直接的经历。

**Units 仅在 [RxCocoa](https://github.com/ReactiveX/RxSwift/tree/master/RxCocoa) 项目中。然而如果有需要，同样的原理可以很轻松的被实现在其他 Rx 的实现中。这里没有私有API的魔法需要**

**Units 是完全可选的。你可以在你程序的任何地方使用原始的 observable 序列，并且所有 RxCocoa 的接口能和 observable 序列兼容使用**

Units 还帮助交流和确认 observable 序列属性穿越接口边界。

当我们在写 Cocoa/UIKit 时，这有一些重要的属性。

* 不能错误退出
* 在主调度器上观察
* 在主调度器上订阅
* 分享副作用

## How they work

在它的核心中，它只是一个引用 observable 序列的结构。

你可以把他们想成一种 observable 序列的构造模型。当一个序列被构造，调用 `.asObservable()` 将转换一个 unit 成为一个 observable 序列。

## Why they are named Units

使用一对类比将会帮助我们解释不常见的理念。下面是一些类比告诉我们 units 在物理世界和 RxCocoa（Rx units）之间的相似点。

Analogies:

| 物理 units                      | Rx units                                                            |
|-------------------------------------|---------------------------------------------------------------------|
| 数字 (一个值)                  | observable 序列 (序列的值)                            |
| 空间单位 (m, s, m/s, N ...) | Swift 结构体 (Driver, ControlProperty, ControlEvent, Variable, ...) |

一个物理单位是一对数字和其匹配的空间单位。<br/>
一个Rx单位（unit）是一对一个 observable 序列和其匹配的描述 observable 序列属性的结构体。

当使用物理单位时，数字是一个基本的组成胶水：比如实数和复数。<br/>
当我们使用 Rx 单位时，Observable 序列是一个基本的组成胶水。

物理单位和 [dimensional analysis](https://en.wikipedia.org/wiki/Dimensional_analysis#Checking_equations_that_involve_dimensions) 可以在复杂计算中减缓一些错误级别。


数字有一些操作符: `+`, `-`, `*`, `/`.<br/>
Observable 序列也有一些操作符: `map`, `filter`, `flatMap` ...

物理单位通过使用相应的数字操作定义操作符，比如：

`/` 操作在物理单位被定义为使用 `/` 操作数字。

11 m / 0.5 s = ...
* 首先，转换到**数字**单位，并且**应用** `/` **操作符** `11 / 0.5 = 22`
* 然后，计算单位（m / s）
* 最后，组合结果 = 22 m / s

Rx 单位通过使用一致的 observable 序列操作符定义操作（这就是内部操作符如何运行的），例如

`Driver` 上的 `map` 操作符被定义为在其 observable 序列上使用  `map` 操作符。

```swift
let d: Driver<Int> = Drive.just(11)
driver.map { $0 / 0.5 } = ...
```

* 首先，转换 `Driver` 为 **observable 序列** 并且 **使用** `map` **操作符**
```swift
let mapped = driver.asObservable().map { $0 / 0.5 } // this `map` is defined on observable sequence
```

* 然后，组合结果来获得单位值
```swift
let result = Driver(mapped)
```

在物理中有一系列基本单位[(`m`, `kg`, `s`, `A`, `K`, `cd`, `mol`)](https://en.wikipedia.org/wiki/SI_base_unit)是正交的。<br/>
在 `RxCocoa` 中有一系列有趣的 observable 序列基础属性是正交的。
	
	* 不能错误退出
	* 在主调度器上观察
	* 订阅在主调度器上
	* 分享副作用


物理中派生的单位有时有专有的名称。<br/>
例如：
```
N (牛顿) = kg * m / s / s
C (库仑) = A * s
T (特斯拉) = kg / A / s / s
```

Rx 派生的单位也有专有的名称。<br/>
例如：
```
Driver = (不能错误退出) * (在主调度器上观察) * (分享副作用)
ControlProperty = (分享副作用) * (订阅在主调度器上)
Variable = (不能错误退出) * (分享副作用)
```

物理中转换不同的单位通过在数学上的操作符定义的 `*`, `/`。<br/>
Rx 中转换不同的单位通过 observable 序列操作符的帮助。

例如：
```
不能错误退出 = catchError
在主调度器上观察 = observeOn(MainScheduler.instance)
订阅在主调度器上 = subscribeOn(MainScheduler.instance)
分享副作用 = share* (one of the `share` operators)
```

## RxCocoa units

### Driver unit 驱动单元

* 不能错误退出
* 在主调度器上观察
* 分享副作用 (`shareReplayLatestWhileConnected`)

### ControlProperty / ControlEvent  控制属性 / 控制事件

* 不能错误退出
* 在主调度器上订阅
* 在主调度器上观察
* 分享副作用

### 变量 (Variable)

* 不能错误退出
* 分享副作用

## 驱动 (Driver)

这是非常精心计划的单元。它的目的是提供一个直接的方式去写 UI 层的响应式代码。

### 为什么被命名为驱动（Driver）

它的目标使用场景是模型化驱动你应用的序列。

例如：
* 从 CoreData 模型驱动 UI
* 使用从其他 UI 元素的值驱动 UI （bindings）
...


类似普通的操作系统驱动，假如一个序列错误退出， 你的应用将停止用户输入的相应。

那些被观察在主线程的元素还非常的重要因为 UI 元素和应用逻辑通常是线程不安全的。


另外， `驱动(Drive)` 单元构建了一个 observable 分享副作用的序列。

E.g.


### Practical usage example  实际用例

这是一个典型的开始例子。

```swift
let results = query.rx_text
    .throttle(0.3, scheduler: MainScheduler.instance)
    .flatMapLatest { query in
        fetchAutoCompleteItems(query)
    }

results
    .map { "\($0.count)" }
    .bindTo(resultCount.rx_text)
    .addDisposableTo(disposeBag)

results
    .bindTo(resultsTableView.rx_itemsWithCellIdentifier("Cell")) { (_, result, cell) in
        cell.textLabel?.text = "\(result)"
    }
    .addDisposableTo(disposeBag)
```

这个代码的意图是：

* Throttle 用户输入
* 联系服务器并且获取一个用户列表（每一个请求）
* 绑定结果到两个 UI 元素：显示结果到 tableView 和显示有多少条记录的标签。


那么，这个代码有什么问题？

* 如果 `fetchAutoCompleteItems` observable 序列错误退出（连接失败或者处理错误），这个错误将会解绑一切并且 UI 不会响应更多的请求。
* 如果 `fetchAutoCompleteItems` 在一些后台线程返回结果，结果会从后台线程被绑定到 UI 元素，而这会引起不确定的奔溃。
* 结果被绑定到两个 UI 元素，这意味着每个用户查询会制造两个HTTP请求对应两个 UI 元素，这并不是预期的行为。


更合适的版本的代码应该看起来像如下：

```swift
let results = query.rx_text
    .throttle(0.3, scheduler: MainScheduler.instance)
    .flatMapLatest { query in
        fetchAutoCompleteItems(query)
            .observeOn(MainScheduler.instance)  // results are returned on MainScheduler
            .catchErrorJustReturn([])           // in the worst case, errors are handled
    }
    .shareReplay(1)                             // HTTP requests are shared and results replayed
                                                // to all UI elements

results
    .map { "\($0.count)" }
    .bindTo(resultCount.rx_text)
    .addDisposableTo(disposeBag)

results
    .bindTo(resultTableView.rx_itemsWithCellIdentifier("Cell")) { (_, result, cell) in
        cell.textLabel?.text = "\(result)"
    }
    .addDisposableTo(disposeBag)
```


在大型系统中确认所有这些要求被合适的处理是有挑战的，但是使用编译器和单位去证明这些要求被满足会有更简单方法。


下面代码看起来大致一样：

```swift
let results = query.rx_text.asDriver()        // This converts a normal sequence into a `Driver` sequence.
    .throttle(0.3, scheduler: MainScheduler.instance)
    .flatMapLatest { query in
        fetchAutoCompleteItems(query)
            .asDriver(onErrorJustReturn: [])  // Builder just needs info about what to return in case of error.
    }

results
    .map { "\($0.count)" }
    .drive(resultCount.rx_text)               // If there is a `drive` method available instead of `bindTo`,
    .addDisposableTo(disposeBag)              // that means that the compiler has proven that all properties
                                              // are satisfied.
results
    .drive(resultTableView.rx_itemsWithCellIdentifier("Cell")) { (_, result, cell) in
        cell.textLabel?.text = "\(result)"
    }
    .addDisposableTo(disposeBag)
```


所以这里发生了什么？


第一个 `asDriver` 方法转换 `ControlProperty` 单元成为一个 `Driver` 单元。

```swift
query.rx_text.asDriver()
```

注意这不需要任何其他的操作。`驱动（Driver）`有所有 `ControlProperty` 单元的属性，并且添加了其他。下面的 observable 序列只是被包装了作为 `驱动Driver` 单元。

第二个改变是：

```swift
.asDriver(onErrorJustReturn: [])
```

任何 observable 序列都能被转换成 `Driver驱动` 单元，只要它满足3个属性：

* 不能错误退出
* 在主调度器上观察
* 分享副作用(`shareReplayLatestWhileConnected`)

所以你如何确认这些属性被满足？仅使用 Rx 操作。`asDriver(onErrorJustReturn: [])` 等价于如下代码：

```
let safeSequence = xs
  .observeOn(MainScheduler.instance)       // observe events on main scheduler
  .catchErrorJustReturn(onErrorJustReturn) // can't error out
  .shareReplayLatestWhileConnected         // side effects sharing
return Driver(raw: safeSequence)           // wrap it up
```

最终的片段是使用 `drive` 代替使用 `bindTo`。

`drive` 仅在 `Driver驱动` 上被定义。这意味着如果你看见 `drive` 出现在代码中，observable 序列从不错误退出并且它在主线程上观察，这样绑定一个 UI 元素是安全的。

然而，理论上，有人可以定义一个 `drive` 方法工作在 `ObservableType` 上或者另一些接口，所以为了额外的安全，在绑定 UI 元素之前会用 `let results: Driver<[Results]> = ...` 创建一个临时的定义，为了完全验证这是必要的。然而，我们将留给读者去决定这是否是一个现实的情况。
