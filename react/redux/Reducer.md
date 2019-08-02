# Reducer
> type Reducer<S, A> = (state: S, action: A) => S

**Reducer**也称为`reducing function`，**Reducer**并不是Redux特有的函数，而是函数式编程中的一个概念 -- **累加**。如果你觉得很面生，那么有个API你一定很熟：`Array.prototype.reduce`。

**Reducer**本身也是与数组中的reduce方法类似，接收两个参数，上一个累积运算值，与当前被累积的值，处理后返回一个新的值，满足函数式编程的规则。

在Redux中，**Reducer**的上一次累积运算值是被修改前的state对象，被累积的值是action，经过处理最后返回一个新的state对象。

为了满足函数式编程，所以**Reducer**必须是纯函数，也就是相同输入要得到相同的输出，不应该产生任何的副作用。

**注意！在Reducer中不能有任何的API调用**

重点：
- Reducer是纯函数。
- Reducer不能有副作用，不能影响之前已存在的任何的引用。
- Reducer会接收改变之前的state，与一个action，最后返回一个新的state。
- Reducer中不能有API调用。
- 也不能有`时间戳`和`随机数`的使用，这样可能会造成输出不一致。
