## Why

**Rx enables building apps in a declarative way.**
**Rx 以一种声明的方式开发apps**

### 绑定(bindings)

```swift
Observable.combineLatest(firstName.rx_text, lastName.rx_text) { $0 + " " + $1 }
    .map { "Greetings, \($0)" }
    .bindTo(greetingLabel.rx_text)
```

This also works with `UITableView`s and `UICollectionView`s.
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

**Official suggestion is to always use `.addDisposableTo(disposeBag)` even though that's not necessary for simple bindings.**
**关键建议一直使用 `.addDisposableTo(disposeBag)`， 即使在简单绑定并不需要**

### Retries
### 重试

It would be great if APIs wouldn't fail, but unfortunately they do. Let's say there is an API method:
如果 APIs 不会失败，那将非常棒，但是不幸的是他们会失败。比如有如下API方法：

```swift
func doSomethingIncredible(forWho: String) throws -> IncredibleThing
```

If you are using this function as it is, it's really hard to do retries in case it fails. Not to mention complexities modeling [exponential backoffs](https://en.wikipedia.org/wiki/Exponential_backoff). Sure it's possible, but the code would probably contain a lot of transient states that you really don't care about, and it wouldn't be reusable.
如果你正在使用像这样的函数，如果失败了将真的很难重试。更别提复杂模型 [exponential backoffs](https://en.wikipedia.org/wiki/Exponential_backoff)。 当然这是可以做到的，但是代码可能会包含很多真的不该关注的暂态，并且将不好复用。

Ideally, you would want to capture the essence of retrying, and to be able to apply it to any operation.
观念上说，你会想要捕捉重试的本质，并且对任何操作都能够加以应用。

This is how you can do simple retries with Rx
使用 Rx 就是可以做到如此简单

```swift
doSomethingIncredible("me")
    .retry(3)
```

You can also easily create custom retry operators.
你还可以简单的创建自己的重试操作符。

### Delegates
### 派遣

Instead of doing the tedious and non-expressive:
代替使用沉闷的不释义的方式：

```swift
public func scrollViewDidScroll(scrollView: UIScrollView) { [weak self] // what scroll view is this bound to?
    self?.leftPositionConstraint.constant = scrollView.contentOffset.x
}
```

... write
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

Use [`rx_observe` and `rx_observeWeakly`](GettingStarted.md#kvo)
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

### Transient state
### 暂态

There are also a lot of problems with transient state when writing async programs. A typical example is an autocomplete search box.
当在写异步程序时，会有大量暂态的问题。一个典型的例子就是自动完成搜索框。

If you were to write the autocomplete code without Rx, the first problem that probably needs to be solved is when `c` in `abc` is typed, and there is a pending request for `ab`, the pending request gets cancelled. OK, that shouldn't be too hard to solve, you just create an additional variable to hold reference to the pending request.
如果你没有用 Rx 来写自动完成功能，那么第一个可能需要解决的问题就是当 `abc` 中的 `c` 被键入，未返回的 `ab` 的请求将被取消。好吧，这应该不难被解决，你只是创建一个额外的变量来持有这个挂起请求的引用。

The next problem is if the request fails, you need to do that messy retry logic. But OK, a couple more fields that capture the number of retries that need to be cleaned up.
下一个问题是，如果这个请求失败了，你需要重做一遍复杂凌乱的重试逻辑。但是可以，几个更多捕获重试数量的字段需要被情理。

It would be great if the program would wait for some time before firing a request to the server. After all, we don't want to spam our servers in case somebody is in the process of typing something very long. An additional timer field maybe?
如果程序能够在请求服务器之前等待一些时间那就太棒啦。在此之后，假如有人在处理很长的输入时，我们就不用给我们服务器发送垃圾信息。也许需要一个额外的的定时器字段？

There is also a question of what needs to be shown on screen while that search is executing, and also what needs to be shown in case we fail even with all of the retries.
另外还有一个问题就是当在执行搜索时，屏幕上需要显示些什么，并且假如我们甚至在所有的重试都失败之后，需要显示什么。

Writing all of this and properly testing it would be tedious. This is that same logic written with Rx.
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

There are no additional flags or fields required. Rx takes care of all that transient mess.
这里并不需要额外的标记和字段。Rx 管理了所有这些短暂的混乱。

### Compositional disposal
### 组合的清理

Let's assume that there is a scenario where you want to display blurred images in a table view. First, the images should be fetched from a URL, then decoded and then blurred.
让我们假设有一个想法，你需要在 table view 中展示模糊不清的图片。首先图片应该从一个URL中获取，然后解码并且使之模糊。

It would also be nice if that entire process could be cancelled if a cell exits the visible table view area since bandwidth and processor time for blurring are expensive.
如果这整个过程能够被取消那就太棒啦，当一个 cell 退出可见的 table view 区域，因为模糊图片的带宽和处理时间是昂贵的。

It would also be nice if we didn't just immediately start to fetch an image once the cell enters the visible area since, if user swipes really fast, there could be a lot of requests fired and cancelled.
一旦当 cell 进入可视区域，如果我们我们不马上开始获取图片那就太棒啦。因为如果用户滑动真的很快，会产生大量的请求和取消。

It would be also nice if we could limit the number of concurrent image operations because, again, blurring images is an expensive operation.
如果我们能够限定并发的图片操作那就太棒啦，因为再次模糊图片是一个费时的操作。

This is how we can do it using Rx:
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

This code will do all that and, when `imageSubscription` is disposed, it will cancel all dependent async operations and make sure no rogue image is bound to the UI.
这个代码会做上述说的所有操作，当 `imageSubscription` 被清理，这将取消所有依赖的异步操作并且确保没有遗留的图片被绑定在 UI 上。

### Aggregating network requests
### 聚集网络请求

What if you need to fire two requests and aggregate results when they have both finished?
假如你需要出发两个请求，并且当他们都完成时集合他们的结果？

Well, there is of course the `zip` operator
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

So what if those APIs return results on a background thread, and binding has to happen on the main UI thread? There is `observeOn`.
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

There are many more practical use cases where Rx really shines.
这有许多使用 Rx 真的非常厉害的实际用例。

### State
### 状态

Languages that allow mutation make it easy to access global state and mutate it. Uncontrolled mutations of shared global state can easily cause [combinatorial explosion] (https://en.wikipedia.org/wiki/Combinatorial_explosion#Computing).
语言允许变化使之更容易接触全局状态并且改变他。无法控制的共享全局状态的变化会很容易引起 [combinatorial explosion] (https://en.wikipedia.org/wiki/Combinatorial_explosion#Computing)。

But on the other hand, when used in a smart way, imperative languages can enable writing more efficient code closer to hardware.
但是另一方面说，当用一个聪明的方法使用时，命令式语言能够写出更有接近硬件有效率的代码。

The usual way to battle combinatorial explosion is to keep state as simple as possible, and use [unidirectional data flows](https://developer.apple.com/videos/play/wwdc2014-229) to model derived data.
通常对抗 combinatorial explosion 的方法是尽可能地保持状态的简单，并且使用 [单向数据流unidirectional data flows](https://developer.apple.com/videos/play/wwdc2014-229) 去模型化收到的数据。

This is where Rx really shines.
这正是 Rx 真牛逼的地方。

Rx is that sweet spot between functional and imperative worlds. It enables you to use immutable definitions and pure functions to process snapshots of mutable state in a reliable composable way.
Rx 最强的地方是在函数式和命令式时间之间。他能够使你用一种可靠地组合方式使用不变定义和纯函数去处理可变状态的快照。

So what are some practical examples?
所以什么是实际的例子？

### Easy integration
### 易于集成

What if you need to create your own observable? It's pretty easy. This code is taken from RxCocoa and that's all you need to wrap HTTP requests with `NSURLSession`
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

### Benefits
### 好处

In short, using Rx will make your code:
简而言之，使用 Rx 会使你的代码：

* Composable <- Because Rx is composition's nickname
* Reusable <- Because it's composable
* Declarative <- Because definitions are immutable and only data changes
* Understandable and concise <- Raising the level of abstraction and removing transient states
* Stable <- Because Rx code is thoroughly unit tested
* Less stateful <- Because you are modeling applications as unidirectional data flows
* Without leaks <- Because resource management is easy
* 组件化 <- 因为 Rx 是组件化的昵称
* 可复用 <- 因为他是组件化的
* 声明式的 <- 因为定义是不可变的并且只有数据变化
* 可理解并且简洁的 <- 提升抽象的等级并且移除暂态
* 稳定的 <- 因为 Rx 的代码完全的单元测试过
* 更少的有状态的 <- 因为你以单向数据流的方法模型化你的应用
* 没有泄露 <- 因为管理资源很简单

### It's not all or nothing
### 并不是非黑即白

It is usually a good idea to model as much of your application as possible using Rx.
通常尽可能的使用 Rx 去模型化你的应用是一个好点子。

But what if you don't know all of the operators and whether or not there even exists some operator that models your particular case?
但是假如你并不知道所有的操作符并且不知道是否有存在的操作符去模型化你特定的用例。

Well, all of the Rx operators are based on math and should be intuitive.
好吧，所有 Rx 操作符都基于数学并且应该很直观。

The good news is that about 10-15 operators cover most typical use cases. And that list already includes some of the familiar ones like `map`, `filter`, `zip`, `observeOn`, ...
好消息是，大概10-15个操作符覆盖了几乎大部分的典型用例。并且其中已经已经包含一些注明的操作符，例如 `map`, `filter`, `zip`, `observeOn`, ...

There is a huge list of [all Rx operators](http://reactivex.io/documentation/operators.html) and list of all of the [currently supported RxSwift operators](API.md).
这有个很全的列表 [all Rx operators](http://reactivex.io/documentation/operators.html) 并且这有所有当前支持的API [currently supported RxSwift operators](API.md)。

For each operator, there is a [marble diagram](http://reactivex.io/documentation/operators/retry.html) that helps to explain how it works.
对于每个操作符，都有一个 [marble diagram](http://reactivex.io/documentation/operators/retry.html)，这能帮助解释他是怎么工作的。

But what if you need some operator that isn't on that list? Well, you can make your own operator.
但是假如你需要一些操作并不在列表上？那么你可以制造你自己的操作符。

What if creating that kind of operator is really hard for some reason, or you have some legacy stateful piece of code that you need to work with? Well, you've got yourself in a mess, but you can [jump out of Rx monads](GettingStarted.md#life-happens) easily, process the data, and return back into it.
即使处于一些原因，真的很难创建操作符，或者你有一些遗留的有状态的代码片段你需要在工作上使用？那么，你有自己的一个烂摊子，但是你能简单地[跳出 Rx monads](GettingStarted.md#life-happens)，处理数据并且返回。
