# 异步 Action
> type AsyncAction = any

文件名与标题名不一致，是为了文件排序看的舒心一些。

首先要明确一点，**异步 action**是不能被reducer消费的，所以**异步 action**是给dispatching函数用的值。

在**异步 action**给到Base dispatch之前，会先经过一层middleware，这时middleware会将action转换成一个action或者一组action。

**异步 action**可以有多种type，这个就取决于你的middleware了。它通常是Promise或者thunk之类的异步原生数据模型，虽然不会在你给到middleware时立即将数据传给reducer，但是在操作完成回会触发action的分发事件。

重点：
- 异步action不能直接被reducer消费。
- 异步action一般给到dispatching函数，然后会经过一层middleware，然后才到base dispatch。
- 异步action可以有多种的type，取决于用的middleware，当然用什么middleware还是取决于业务需求。
- middleware一般会是Promise和thunk这写异步原生数据模型。