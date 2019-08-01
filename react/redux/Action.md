# Action
> type Action = Object

**Action**是一个`普通对象`，用来表示即将改变`state`的意图，store只提对外供了**Action**这个入口来修改数据，也就是说要把数据放进store中，只能通过**Action**做桥接。

无论任何数据，只要想存入或者更新Store中的`state`，最终都会被dispatch成**Action**，或者说演变成**Action**对象。

因为约定俗成，**Action**必须要有一个`type`字段，它可以说是**Action**的名字，执行者就是通过type找到对应的**Action**对象。`type`可以定义为常量，然后从其他的module导入。不用Symbols的因为String可以被序列化。

除了`type`之外，**Action**对象的结构完全取决于你自己。但是最好遵循一下[Flux Standard Action](https://github.com/redux-utilities/flux-standard-action#flux-standard-action)的`FSA的规则`
```javascript
// normal
{
  type: 'ADD_TODO',
  payload: {
    text: 'Do something.'  
  },
  meta: '任何类型的扩展内容'
}

// error
{
  "type": "ADD_TODO",
  "payload": new Error(),
  "error": "true"
}
```

你甚至还可以从npm下载`flux-standard-action`包，它会给你提供一些判断功能的函数：
- isFSA(action)
- isError(action)

如果满足FSA的规则，或者**Action**错误规则，会给你返回true。

重点：
- action是一个对象。
- 用于告诉Store为什么要修改state。
- 修改数据必须要用action。
- action对象必须有type字段，最好定义为常量，可以被序列化。
- action对象除了type字段以外，其他的数据结构随自己喜欢，如果可以尽量遵循`flux的Action规则`。