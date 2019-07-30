# hooks-useEffect
useEffect钩子会触发3个钩子`componentDidMount`、`componentDidUpdate`、`componentWillUnmount`

`useEffect(fn, array)`，第二个参数是绑定触发更新的列表，绑定state或props，只要更改就触发useEffect，如果写成空数组`[]`说明只会执行一次，最好是协商，否则会一直刷钩子函数。

如果useEffect函数return了一个函数，那么就会在组件销毁`componentWillUnmount`阶段执行