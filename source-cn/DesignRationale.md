设计原理
================

## 为什么错误类型不是泛型

```Swift
enum Event<Element>  {
    case Next(Element)      // next element of a sequence
    case Error(ErrorType)   // sequence failed with error
    case Completed          // sequence terminated successfully
}
```

Let's discuss the pros and cons of `ErrorType` being generic.
让我们来讨论一下，如果 `ErrorType` 是反应的优劣点。

If you have a generic error type, you create additional impedance mismatch between two observables.
如果你有一个错误类型的泛型，你创建两个 observable 之间的额外阻抗失配。

Let's say you have:
比如你有：

`Observable<String, E1>` 和 `Observable<String, E2>`

There isn't much you can do with them without figuring out what the resulting error type will be.
没有多少，你可以与他们无关不搞清楚所产生的错误类型是什么


Will it be `E1`, `E2` or some new `E3` maybe? So, you would need a new set of operators just to solve that impedance mismatch.
它可能是 `E1`, `E2` 或是新的 `E3`？所以你可能需要一组操作符，只是用来解决阻抗失配。

This hurts composition properties, and Rx isn't concerned with why a sequence fails, it just usually forwards failures further down the observable chain.
这破坏了组件属性，并且 Rx 并不关心一个序列为什么失败，他通常只是进一步转发失败下的 observable 链。

There is an additional problem that, in some cases, operators might fail due to some internal error, in which case you wouldn't be able to construct a resulting error and report failure.
这是另一个问题，在某些情况下，操作符也许会因为内部错误失败，在那种情况下，你将不能构建一个结果错误，并且报告失败。

But OK, let's ignore that and assume we can use that to model sequences that don't error out. Could it be useful for that purpose?
不过还好，让我们忽略那个，并且假设我们能用那个区模型化不会错误退出的序列。这会对那个目的有用吗？


Well yes, it potentially could be, but let's consider why you would want to use sequences that don't error out.
好吧是的，它可能会有效，但是让我们想想，你为什么需要使用不会错误退出的序列。

One obvious application would be for permanent streams in the UI layer that drive the entire UI. When you consider that case, it's not really sufficient to only use the compiler to prove that sequences don't error out, you also need to prove other properties. For instance, that elements are observed on `MainScheduler`.
一个平淡无奇的应用是用不变的流来驱动整个UI用户界面。当你考虑那个例子，只用编译器来保证序列不出错退出真的是不够的，你还需要保证其他属性。比如说，元素需要被观察在 `MainScheduler`上。

What you really need is a generic way to prove traits for observable sequences. There are a lot of properties you could be interested in. For example:
你真正需要是一个通用的方法来验证 observable 序列的特性。这有许多属性你可能感兴趣。举个例子：

* sequence terminates in finite time (server side)
* sequence contains only one element (if you are running some computation)
* sequence doesn't error out, never terminates and elements are delivered on main scheduler (UI)
* sequence doesn't error out, never terminates and elements are delivered on main scheduler, and has refcounted sharing (UI)
* sequence doesn't error out, never terminates and elements are delivered on specific background scheduler (audio engine)
* 序列终止于有限的时间内（服务端）
* 序列只包含一个元素（如果你在运行一些计算）
* 序列不错误退出，从不终止并且在主调度器（UI）上接收元素。
* 序列不错误退出，从不终止并且在主调度器上接收元素，而且有引用计数分享（UI）
* 序列不错误退出，从不终止并且在特殊的调度器上接收元素（音频引擎）

What you really want is a general compiler-enforced system of traits for observable sequences, and a set of invariant operators for those wanted properties.
你真的想要的是一个通用的强制编译的 observable 序列特性系统，以及那些想要属性的不变式操作符集合。

A good analogy would be:
一个适当的类比：

```
1, 3.14, e, 2.79, 1 + 1i      <->    Observable<E>
1m/s, 1T, 5kg, 1.3 pounds     <->    Errorless observable, UI observable, Finite observable ...
```

There are many ways to do such a thing in Swift by either using composition or inheritance of observables.
在 Swift 中可以用很多方式实现这个事情，比如使用组合或者继承 observables。

An additional benefit of using a unit system is that you can prove that UI code is executing on the same scheduler and thus use lockless operators for all transformations.
使用单位系统的另一个好处是你可以验证 UI 的代码在同一个调度器(scheduler)上执行，然后在所有转变中使用无所操作。

Since RxSwift already doesn't have locks for single sequence operations, and all of the locks that do exist are in stateful components (e.g. UI), there are practically no locks in RxSwift code, which allows for such details to be compiler enforced.
因为 RxSwift 的单序列操作符已经没有锁，所有的锁存在于稳定的组件中（例如，UI），实际上 RxSwift 的代码中不含有锁，因此允许强制编译。

There really is no benefit to using typed Errors that couldn't be achieved in other cleaner ways while preserving Rx compositional semantics.
当保留 Rx 组件的语义时使用 Error 泛型是真的没有好处并不能达到一个清晰的方式。
