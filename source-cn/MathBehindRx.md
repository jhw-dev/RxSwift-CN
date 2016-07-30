Rx下的数学
==============

## Observer 和 迭代器（Iterator） / 枚举器（Enumerator）/ 生成器（Generator） / 序列（sequences） 之间的二元性

There is a duality between the observer and generator patterns. This is what enables us to transition from the async callback world to the synchronous world of sequence transformations.
observer 和生成器模式之间存在二元性。这正是使我们从异步回调世界到同步的序列世界的转换。

In short, the enumerator and observer patterns both describe sequences. It's fairly obvious why the enumerator defines a sequence, but the observer is slightly more complicated.
简而言之，枚举器和观察者模式两者都描述序列。这相当明显为什么枚举器定义一个序列，但是观察者稍微更复杂些。

There is, however, a pretty simple example that doesn't require a lot of mathematical knowledge. Assume that you are observing the position of your mouse cursor on screen at given times. Over time, these mouse positions form a sequence. This is, in essence, an observable sequence.
然而，这有一个非常简单不需要大量数学知识的例子。假设你正在观察在给定屏幕上你鼠标光标的位置。本质上，这是一个 observable 序列。

There are two basic ways elements of a sequence can be accessed:
这有2个获取元素序列的基本方式：

* Push interface - Observer (observed elements over time make a sequence)
* 推送接口 - 观察者（被观察的元素一段时间制造一个序列）
* Pull interface - Iterator / Enumerator / Generator
* 拉取接口 - 迭代器 / 枚举器 / 生成器

You can also see a more formal explanation in this video:
你还可以在视频中查看更多正式解释：

* [Expert to Expert: Brian Beckman and Erik Meijer - Inside the .NET Reactive Framework (Rx) (video)](https://www.youtube.com/watch?v=looJcaeboBY)
* [Reactive Programming Overview (Jafar Husain from Netflix)](https://www.youtube.com/watch?v=dwP1TNXE6fc)
