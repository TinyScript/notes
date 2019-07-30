# React注意项

### 16.8+的更新
- hooks
- 函数式组件
- 以后会慢慢移除类组件等，会以函数式编程为主
- render返回值新增fragment标签，<></>
- render返回值新增了strings
- React.Component新增了纯组件封装React.PureComponent，会自动处理shouldComponentUpdate的钩子。

--- 

### 注意点记录
- 如果子组件更新，父组件也会跟着更新，跟vue会自动去校验选择性更新不一样。
- 如果生命周期没有用对，组件会发生大量的重复性渲染。
- this的指向需要开发者自己bind，否则默认是undefined。
- jsx的问题，必须要用div进行包裹最外层。react新版本处理了这个问题。
- 什么时候能动state，生命周期在Will的时候可以动，render也可以动，shouldComponentUpdate和componentWillUpdate，15版本。
- 