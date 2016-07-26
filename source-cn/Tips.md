Tips
====

* 总是力求为你的系统或其他部分构建模型，成为纯函数(pure functions)。纯函数易于被测试并且能被用来修改操作符的行为。
* 当你在使用 Rx，首先尝试使用组合内置操作符。
* 如果你经常使用一些操作符的组合，那么封装你自己的操作符

例如：

```swift
extension ObservableType where E: MaybeCool {

    @warn_unused_result(message="http://git.io/rxs.uo")
    public func coolElements()
        -> Observable<E> {
          return filter { e -> Bool in
              return e.isCool
          }
    }
}
```

  * Rx 操作符尽可能的通用，但是总有边缘用例难以作模型。如果是这样，你可以创建你自己的操作符并且可能使用内置的操作符作为参考。

  * 总是使用操作符去构成订阅。

  **避免嵌套定订阅调用。例如：**

  ```swift
  textField.rx_text.subscribeNext { text in
      performURLRequest(text).subscribeNext { result in
          ...
      }
      .addDisposableTo(disposeBag)
  }
  .addDisposableTo(disposeBag)
  ```

  **更好的方式是使用操作符链式调用**

  ```swift
  textField.rx_text
      .flatMapLatest { text in
          // Assuming this doesn't fail and returns result on main scheduler,
          // otherwise `catchError` and `observeOn(MainScheduler.instance)` can be used to
          // correct this.
          return performURLRequest(text)
      }
      ...
      .addDisposableTo(disposeBag) // only one top most disposable
  ```
