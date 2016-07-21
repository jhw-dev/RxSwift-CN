Linux
=====

我们为 Linux 做了一个试验。

为了测试它，在你的测试目录中按如下内容创建 `Package.swift`：

```
import PackageDescription

let package = Package(
    name: "MyShinyUnicornCat",
    dependencies: [
        .Package(url: "https://github.com/ReactiveX/RxSwift.git", Version(2, 0, 0))
    ]
)
```

有哪些是可以工作的：
* 使用 Swift 包管理分配
* 单线程模式 （当前线程调度）
* 一般的单元测试通过的。
* 项目能被编译和“使用”：
	* RxSwift
	* RxBlocking
	* RxTests

有哪些是不工作的：
* 调度器 - 因为他们依赖 https://github.com/apple/swift-corelibs-libdispatch 并且它还没被发布
* 多线程 - 依然不能使用 c11 锁
* 当在 `Linux` 使用 `ErrorType` 出于某些原因它看起来像 Swift 编译器生成了错误的代码， 所以不能使用 errors， 否则你会得到很奇怪的崩溃。
