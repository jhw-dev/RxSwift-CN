Warnings
========

### 未使用的disposeable(unused-disposable)

下面对于返回 `Disposable` 的 `subscribe*`, `bind*` 和 `drive*` 家族的函数是有效的。

你将会受到一个警告，因为做了一些下面的这些：

```Swift
let xs: Observable<E> ....

xs
  .filter { ... }
  .map { ... }
  .switchLatest()
  .subscribe(onNext: {
    ...
  }, onError: {
    ...
  })
```

返回一个订阅 `Disposable` 的 `subscribe` 的函数能被用来取消计算和释放资源。然而，不使用它（并且不 dispose 它）将引起一个错误

终止这些流利调用的更好方法是通过使用 `DisposeBag`，或通过链式调用 `.addDisposableTo(disposeBag)` 或通过直接给包增加 disposable。

```Swift
let xs: Observable<E> ....
let disposeBag = DisposeBag()

xs
  .filter { ... }
  .map { ... }
  .switchLatest()
  .subscribe(onNext: {
    ...
  }, onError: {
    ...
  })
  .addDisposableTo(disposeBag) // <--- note `addDisposableTo`
```

当 `disposeBag` 被释放时，在 disposables 中的会被自动释放。

`xs`在无论是` Completed`或` Error`消息可预见的方式终止的情况下，不处理订阅 `Disposable` 不会造成任何资源的泄露。 然而，即使在这种情况下，使用处置袋仍是处理订阅一次性的优选方式。它确保元素计算总是在可预测的时刻终止，使代码健壮和未来的证明，因为资源将被适当设置，即使实施` xs`变化。

另一种方式来确保订阅和资源都依赖于某些对象的生命周期是使用` takeUntil`操作符。

```Swift
let xs: Observable<E> ....
let someObject: NSObject  ...

_ = xs
  .filter { ... }
  .map { ... }
  .switchLatest()
  .takeUntil(someObject.rx_deallocated) // <-- note the `takeUntil` operator
  .subscribe(onNext: {
    ...
  }, onError: {
    ...
  })
```

如果忽略订阅 `Disposable` 是所期望的行为，这是如何去除编译器警告。

```Swift
let xs: Observable<E> ....

_ = xs // <-- note the underscore
  .filter { ... }
  .map { ... }
  .switchLatest()
  .subscribe(onNext: {
    ...
  }, onError: {
    ...
  })
```

### 未使用的observable(unused-observable)

你像下面这么做就收到一个警告：

```Swift
let xs: Observable<E> ....

xs
  .filter { ... }
  .map { ... }
```

此代码定义过滤并从 `xs` 序列映射的 observable， 但忽略结果

由于该代码只是定义了一个可观察序列（observable），然后忽略它，它实际上并没有做任何事情。

你的意图可能是要么存储可观察序列（observable）定义并在以后使用它...

```Swift
let xs: Observable<E> ....

let ys = xs // <--- names definition as `ys`
  .filter { ... }
  .map { ... }
```

... 或开始基于定义的计算

```Swift
let xs: Observable<E> ....
let disposeBag = DisposeBag()

xs
  .filter { ... }
  .map { ... }
  .subscribeNext { nextElement in  // <-- note the `subscribe*` method
    // use the element
    print(nextElement)
  }
  .addDisposableTo(disposeBag)
```
