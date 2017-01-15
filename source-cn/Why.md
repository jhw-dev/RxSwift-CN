## Why

**Rx 以一种声明的方式开发apps**

### 绑定(bindings)

```swift
Observable.combineLatest(firstName.rx_text, lastName.rx_text) { $0 + " " + $1 }
    .map { "Greetings, \($0)" }
    .bindTo(greetingLabel.rx_text)
```

`UITableView`s 和 `UICollectionView` 也适用。

```swift
viewModel
    .rows
    .bindTo(resultsTableView.rx_itemsWithCellIdentifier("WikipediaSearchCell", cellType: WikipediaSearchCell.self)) { (_, viewModel, cell) in
        cell.title = viewModel.title
        cell.url = viewModel.url
    }
    .addDisposableTo(disposeBag)
```

**关键建议一直使用 `.addDisposableTo(disposeBag)`， 即使在简单绑定并不需要**

### 重试

如果 APIs 不会失败，那将非常棒，但是不幸的是他们会失败。比如有如下API方法：

```swift
func doSomethingIncredible(forWho: String) throws -> IncredibleThing
```

如果你正在使用像这样的函数，如果失败了将真的很难重试。更别提复杂模型 [exponential backoffs](https://en.wikipedia.org/wiki/Exponential_backoff)。 当然这是可以做到的，但是代码可能会包含很多真的不该关注的暂态，并且将不好复用。

观念上说，你会想要捕捉重试的本质，并且对任何操作都能够加以应用。

使用 Rx 就是可以做到如此简单

```swift
doSomethingIncredible("me")
    .retry(3)
```

你还可以简单的创建自己的重试操作符。

### 派遣

代替使用沉闷的不释义的方式：

```swift
public func scrollViewDidScroll(scrollView: UIScrollView) { [weak self] // what scroll view is this bound to?
    self?.leftPositionConstraint.constant = scrollView.contentOffset.x
}
```

而使用：

```swift
self.resultsTableView
    .rx_contentOffset
    .map { $0.x }
    .bindTo(self.leftPositionConstraint.rx_constant)
```

### KVO

Instead of:

```
`TickTock` was deallocated while key value observers were still registered with it. Observation info was leaked, and may even become mistakenly attached to some other object.
```

and

```objc
-(void)observeValueForKeyPath:(NSString *)keyPath
                     ofObject:(id)object
                       change:(NSDictionary *)change
                      context:(void *)context
```

使用 [`rx_observe` 和 `rx_observeWeakly`](GettingStarted.md#kvo)

This is how they can be used:

```swift
view.rx_observe(CGRect.self, "frame")
    .subscribeNext { frame in
        print("Got new frame \(frame)")
    }
```

or

```swift
someSuspiciousViewController
    .rx_observeWeakly(Bool.self, "behavingOk")
    .subscribeNext { behavingOk in
        print("Cats can purr? \(behavingOk)")
    }
```

### Notifications

Instead of using:

```swift
@available(iOS 4.0, *)
public func addObserverForName(name: String?, object obj: AnyObject?, queue: NSOperationQueue?, usingBlock block: (NSNotification) -> Void) -> NSObjectProtocol
```

... just write

```swift
NSNotificationCenter.defaultCenter()
    .rx_notification(UITextViewTextDidBeginEditingNotification, object: myTextView)
    .map { /*do something with data*/ }
    ....
```

### 暂态

当在写异步程序时，会有大量暂态的问题。一个典型的例子就是自动完成搜索框。

如果你没有用 Rx 来写自动完成功能，那么第一个可能需要解决的问题就是当 `abc` 中的 `c` 被键入，未返回的 `ab` 的请求将被取消。好吧，这应该不难被解决，你只是创建一个额外的变量来持有这个挂起请求的引用。

下一个问题是，如果这个请求失败了，你需要重做一遍复杂凌乱的重试逻辑。但是可以，几个更多捕获重试数量的字段需要被情理。

如果程序能够在请求服务器之前等待一些时间那就太棒啦。在此之后，假如有人在处理很长的输入时，我们就不用给我们服务器发送垃圾信息。也许需要一个额外的的定时器字段？

另外还有一个问题就是当在执行搜索时，屏幕上需要显示些什么，并且假如我们甚至在所有的重试都失败之后，需要显示什么。

写出所有这些逻辑并且测试他将会是沉闷枯燥的。下面是同样的逻辑使用 Rx 写的：

```swift
searchTextField.rx_text
    .throttle(0.3, scheduler: MainScheduler.instance)
    .distinctUntilChanged()
    .flatMapLatest { query in
        API.getSearchResults(query)
            .retry(3)
            .startWith([]) // clears results on new search term
            .catchErrorJustReturn([])
    }
    .subscribeNext { results in
      // bind to ui
    }
```

这里并不需要额外的标记和字段。Rx 管理了所有这些短暂的混乱。

### 组合的清理

让我们假设有一个想法，你需要在 table view 中展示模糊不清的图片。首先图片应该从一个URL中获取，然后解码并且使之模糊。

如果这整个过程能够被取消那就太棒啦，当一个 cell 退出可见的 table view 区域，因为模糊图片的带宽和处理时间是昂贵的。

一旦当 cell 进入可视区域，如果我们我们不马上开始获取图片那就太棒啦。因为如果用户滑动真的很快，会产生大量的请求和取消。

如果我们能够限定并发的图片操作那就太棒啦，因为再次模糊图片是一个费时的操作。

我们使用 Rx 可以这么做：

```swift
// this is a conceptual solution
let imageSubscription = imageURLs
    .throttle(0.2, scheduler: MainScheduler.instance)
    .flatMapLatest { imageURL in
        API.fetchImage(imageURL)
    }
    .observeOn(operationScheduler)
    .map { imageData in
        return decodeAndBlurImage(imageData)
    }
    .observeOn(MainScheduler.instance)
    .subscribeNext { blurredImage in
        imageView.image = blurredImage
    }
    .addDisposableTo(reuseDisposeBag)
```

这个代码会做上述说的所有操作，当 `imageSubscription` 被清理，这将取消所有依赖的异步操作并且确保没有遗留的图片被绑定在 UI 上。

### 聚集网络请求

假如你需要出发两个请求，并且当他们都完成时集合他们的结果？

好吧，那毫无疑问就是 `zip` 操作符了

```swift
let userRequest: Observable<User> = API.getUser("me")
let friendsRequest: Observable<Friends> = API.getFriends("me")

Observable.zip(userRequest, friendsRequest) { user, friends in
    return (user, friends)
}
.subscribeNext { user, friends in
    // bind them to the user interface
}
```

那么假如那些 API 从后台线程中返回，并且必须在主 UI 线程中发生绑定？这就是 `observeOn`

```swift
let userRequest: Observable<User> = API.getUser("me")
let friendsRequest: Observable<[Friend]> = API.getFriends("me")

Observable.zip(userRequest, friendsRequest) { user, friends in
    return (user, friends)
}
.observeOn(MainScheduler.instance)
.subscribeNext { user, friends in
    // bind them to the user interface
}
```

这有许多使用 Rx 真的非常厉害的实际用例。

### 状态

语言允许变化使之更容易接触全局状态并且改变他。无法控制的共享全局状态的变化会很容易引起 [combinatorial explosion] (https://en.wikipedia.org/wiki/Combinatorial_explosion#Computing)。

但是另一方面说，当用一个聪明的方法使用时，命令式语言能够写出更有接近硬件有效率的代码。

通常对抗 combinatorial explosion 的方法是尽可能地保持状态的简单，并且使用 [单向数据流unidirectional data flows](https://developer.apple.com/videos/play/wwdc2014-229) 去模型化收到的数据。

这正是 Rx 真牛逼的地方。

Rx 最强的地方是在函数式和命令式时间之间。他能够使你用一种可靠地组合方式使用不变定义和纯函数去处理可变状态的快照。

所以什么是实际的例子？

### 易于集成

假如你需要创建你自己的 observable? 那是非常的简单。下面的代码是源自 RxCocoa，并且这是你封装 `NSURLSession` 的 HTTP 请求需要的所有动作。

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

### 好处

简而言之，使用 Rx 会使你的代码：

* 组件化 <- 因为 Rx 是组件化的昵称
* 可复用 <- 因为他是组件化的
* 声明式的 <- 因为定义是不可变的并且只有数据变化
* 可理解并且简洁的 <- 提升抽象的等级并且移除暂态
* 稳定的 <- 因为 Rx 的代码完全的单元测试过
* 更少的有状态的 <- 因为你以单向数据流的方法模型化你的应用
* 没有泄露 <- 因为管理资源很简单

### 并不是非黑即白

通常尽可能的使用 Rx 去模型化你的应用是一个好点子。

但是假如你并不知道所有的操作符并且不知道是否有存在的操作符去模型化你特定的用例。

好吧，所有 Rx 操作符都基于数学并且应该很直观。

好消息是，大概10-15个操作符覆盖了几乎大部分的典型用例。并且其中已经已经包含一些注明的操作符，例如 `map`, `filter`, `zip`, `observeOn`, ...

这有个很全的列表 [all Rx operators](http://reactivex.io/documentation/operators.html) 并且这有所有当前支持的API [currently supported RxSwift operators](API.md)。

对于每个操作符，都有一个 [marble diagram](http://reactivex.io/documentation/operators/retry.html)，这能帮助解释他是怎么工作的。

但是假如你需要一些操作并不在列表上？那么你可以制造你自己的操作符。

即使处于一些原因，真的很难创建操作符，或者你有一些遗留的有状态的代码片段你需要在工作上使用？那么，你有自己的一个烂摊子，但是你能简单地[跳出 Rx monads](GettingStarted.md#life-happens)，处理数据并且返回。
