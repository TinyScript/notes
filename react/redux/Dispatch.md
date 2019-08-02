# Dispatch
> type BaseDispatch = (a: Action) => Action
> type Dispatch = (a: Action | AsyncAction) => any

dispatch是一个接收`action`或者`异步 action`的函数，该函数分发action给Store的规则有两种：
- 要么一个或多个action。
- 要么不分发任何action。

我们要分清一般的dispatch与Store提供的没有`middleware`的Base dispatch之间的区别。

- Base dispatch：总是同步的把action从`上一次store返回的state`发往`reducer`，然后得出新的state给到store。
它期望的action是一个可以被`reducer消费`的普通对象。
- Middleware dispatch：经过`Middleware封装的base dispatch`，允许dispatch处理action之外的`异步 action`。
Middleware可以改变、延迟、忽略action或者异步action，也可以在下一个middleware之前对他们进行解释。