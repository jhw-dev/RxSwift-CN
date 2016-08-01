Warnings
========

### <a name="unused-disposable"></a>Unused disposable 未使用的disposeable(unused-disposable)

The following is valid for the `subscribe*`, `bind*` and `drive*` family of functions that return `Disposable`.
下面对于返回 `Disposable` 的 `subscribe*`, `bind*` 和 `drive*` 家族的函数是有效的。

You will receive a warning for doing something such as this:
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

The `subscribe` function returns a subscription `Disposable` that can be used to cancel computation and free resources.  However, not using it (and thus not disposing it) will result in an error.
返回一个订阅 `Disposable` 的 `subscribe` 的函数能被用来取消计算和释放资源。然而，不使用它（并且不 dispose 它）将引起一个错误

The preferred way of terminating these fluent calls is by using a `DisposeBag`, either through chaining a call to `.addDisposableTo(disposeBag)` or by adding the disposable directly to the bag.
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

When `disposeBag` gets deallocated, the disposables contained within it will be automatically disposed as well.
当 `disposeBag` 被释放时，在 disposables 中的会被自动释放。

In the case where `xs` terminates in a predictable way with either a `Completed` or `Error` message, not handling the subscription `Disposable` won't leak any resources. However, even in this case, using a dispose bag is still the preferred way to handle subscription disposables. It ensures that element computation is always terminated at a predictable moment, and makes your code robust and future proof because resources will be properly disposed even if the implementation of `xs` changes.
`xs`在无论是` Completed`或` Error`消息可预见的方式终止的情况下，不处理订阅 `Disposable` 不会造成任何资源的泄露。 然而，即使在这种情况下，使用处置袋仍是处理订阅一次性的优选方式。它确保元素计算总是在可预测的时刻终止，使代码健壮和未来的证明，因为资源将被适当设置，即使实施` xs`变化。

Another way to make sure subscriptions and resources are tied to the lifetime of some object is by using the `takeUntil` operator.
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

If ignoring the subscription `Disposable` is the desired behavior, this is how to silence the compiler warning.
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

### <a name="unused-observable"></a>Unused observable sequence 未使用的observable(unused-observable)

You will receive a warning for doing something such as this:
你像下面这么做就收到一个警告：

```Swift
let xs: Observable<E> ....

xs
  .filter { ... }
  .map { ... }
```

This code defines an observable sequence that is filtered and mapped from the `xs` sequence but then ignores the result.
此代码定义过滤并从 `xs` 序列映射的 observable， 但忽略结果

Since this code just defines an observable sequence and then ignores it, it doesn't actually do anything.
由于该代码只是定义了一个可观察序列（observable），然后忽略它，它实际上并没有做任何事情。

Your intention was probably to either store the observable sequence definition and use it later ...
你的意图可能是要么存储可观察序列（observable）定义并在以后使用它...

```Swift
let xs: Observable<E> ....

let ys = xs // <--- names definition as `ys`
  .filter { ... }
  .map { ... }
```

... or start computation based on that definition
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
