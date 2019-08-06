# Middleware
> type MiddlewareAPI = { dispatch: Dispatch, getState: () => State }
> type Middleware = (api: MiddlewareAPI) => (next: Dispatch) => Dispatch

Middleware 是一个组合dispatch的高阶函数，返回一个新的dispatch。通常将异步 action转还成action。

Middleware 利用复合函数可以让它组合其他的函数，可以用于记录action日志、产生其他诸如变化路由的副作用，或将异步的API调用变为一组同步的action。

具体的详情可以看[applyMiddleware](https://cn.redux.js.org/docs/api/applyMiddleware.html)