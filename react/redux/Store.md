# Store
```javascript
type Store = {
  dispatch: Dispatch
  getState: () => State
  subscribe: (listener: () => void) => () => void
  replaceReducer: (reducer: Reducer) => void
}
```

Store维持这应用的state tree对象，因为应用的构建发生于reducer，所以一个Redux应用应当只有一个Store。

- `dispatch(action)` 是上述的 base dispatch function。
- `getState()` 返回当前store的state。
- `subscribe(listener)` 注册一个state发生变化时的回调函数。
- `replaceReducer(nextReducer)` 可用于热重载和代码分割。


