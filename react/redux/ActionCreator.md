# Action Creator
> type ActionCreator = (...args: any) => Action | AsyncAction

**Action Creator**这个概念就相对比较简单了，它就是一个工厂的概念，调用一次就创建一个action。

官方明确说明不要混淆action和**action Creator**的概念，action是一个信息载体，**action Creator**则是一个生成action的工厂。

调用**action Creator**只会产生action，不会对action进行分发并修改，你想要分发还是得执行Store中的dispatch函数才能进行对应的修改。有时候讲的`bound action creator`概念，指的是一个函数调用了**action creator**并立即将结果分发给一个特定的store实例。

如果**action creator**需要`读取`当前的state、调用API、或引起诸如路由变化等，存在副作用的内容，那么就应该返回一个`异步 action`而不是`action`。

重点：
- action creator是一个生产action的工厂，action是一个数据载体。
- action creator只能生产action，不会对分发进行操作。
- action creator内部如果有存在`副作用的操作`或者`异步操作`，请返回一个`异步 action`。
- `bound action creator`是指创建了结果并立即对一个特定的store实例进行分发