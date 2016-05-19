## RxSwift 与 ReactiveCocoa 的对比

由于 ReactiveCocoa 从 Rx 系列中借鉴了大量的概念，所以在某种程度上 RxSwift 和 ReactiveCocoa 是非常相似的。

本项目其中的一个主要目标是建立一套尽可能简单的 Rx 接口，同时和其他的的 Rx 实现一样，提供了丰富的并发模型，更多的优化能力以及能够紧密的和 Swift 错误处理机制整合。

我们同时也决定不引入任何第三方的依赖，保证本项目仅仅依赖于 Swift/llvm 的编译器。

相对来说，与 ReactiveCoca 最大的不同应该是在与项目在对抽象层的设计上使用了完全不同的思想。

RxSwift 的主要目的是通过可观测序列的形式去提供一个环境无关可组合的计算型抽象胶水层。
然后我们再专注于去提高 RxSwift 在一些具体平台上的体验。为了完成这一目标，RxCocoa 使用计算型泛型抽象来构造更为实用的抽象层并且对 Foundation/Cocoa/UKit 的框架都进行一层封装。这意味着，在 RxSwift 通过提供 `Driver`,`ControlProperty`,`ControlEvent` 等来实现了其他库通过使用计算型泛型的语法来提供相应的接口。
One of the benefits to representing all of these abstractions as a single concept - ​_observable sequences_​ - is that all computation abstractions built on top of them are also composable in the same fundamental way. They all follow the same contract and implement the same interface.
 It is also easy to create flexible subscription (resource) sharing strategies or use one of the built-in ones: `share`, `shareReplay`, `publish`, `multicast`, `shareReplayLatestWhileConnected`...

This library also offers a fine-tunable concurrency model. If concurrent schedulers are used, observable sequence operators will preserve sequence properties. The same observable sequence operators will also know how to detect and optimally use known serial schedulers. ReactiveCocoa has a more limited concurrency model and only allows serial schedulers.

Multithreaded programming is really hard and detecting non trivial loops is even harder. That's why all operators are built in a fault tolerant way. Even if element generation occurs during element processing (recursion), operators will try to handle that situation and prevent deadlocks. This means that in the worst possible case programming error will cause stack overflow, but users won't have to manually kill the app, and you will get a crash report in error reporting systems so you can find and fix the problem.
