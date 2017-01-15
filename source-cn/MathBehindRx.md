Rx下的数学
==============

## Observer 和 迭代器（Iterator） / 枚举器（Enumerator）/ 生成器（Generator） / 序列（sequences） 之间的二元性

observer 和生成器模式之间存在二元性。这正是使我们从异步回调世界到同步的序列世界的转换。

简而言之，枚举器和观察者模式两者都描述序列。这相当明显为什么枚举器定义一个序列，但是观察者稍微更复杂些。

然而，这有一个非常简单不需要大量数学知识的例子。假设你正在观察在给定屏幕上你鼠标光标的位置。本质上，这是一个 observable 序列。

这有2个获取元素序列的基本方式：

* 推送接口 - 观察者（被观察的元素一段时间制造一个序列）
* 拉取接口 - 迭代器 / 枚举器 / 生成器

你还可以在视频中查看更多正式解释：

* [Expert to Expert: Brian Beckman and Erik Meijer - Inside the .NET Reactive Framework (Rx) (video)](https://www.youtube.com/watch?v=looJcaeboBY)
* [Reactive Programming Overview (Jafar Husain from Netflix)](https://www.youtube.com/watch?v=dwP1TNXE6fc)
