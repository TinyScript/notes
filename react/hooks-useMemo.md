# hooks-useMemo
带`use`开头的api都叫hooks

useMemo是memo的加强版，memo是只接收一个组件，useMemo是可以接收一个函数，并且返回一个新的函数。

当你遇到改变父组件时，子组件也会触发重复渲染，那就要考虑使用React.memo了。

```javascript
import React from "react";
const { memo, useState } = React;
const Counter = memo(props => {
  console.log("组件渲染");
  return <h1>{props.data}</h1>;
});
export default function App() {
  const [count, setCount] = useState(0);
  const data = "京程一灯";
  return (
    <>
      <span>{count}</span>
      <input
        type="button"
        onClick={() => setCount(count + 1)}
        value="修改count"
      />
      <Counter data={data} />
    </>
  );
}
```

如果遇到memo不管用时，比如定义了一个click事件然后绑定到子组件中，因为每次更新父组件，也会重新定义函数，这时函数的内存地址变了，子组件用了memo也没有用，所以就要用到useMemo，当然你也可以用useCallback。

useMemo函数第二个参数可以设置update视图的条件，与state或props绑定。
```javascript
import React from "react";
const { memo, useEffect, useCallback, useMemo, useState } = React;
const Counter = memo(props => {
  // useEffect(() => {
  //   console.log('子组件渲染副作用')
  // })
  console.log("子组件渲染");
  return <h1 onClick={props.onClick}>1</h1>;
});
export default function App() {
  // console.log("父亲组件渲染");
  const [count, setCount] = useState(0);
  const onClick = () => {
    console.log("Click");
  };
  // const data = "京程一灯";
  // useEffect(() => {
  //   console.log('父组件副作用')
  // })
  // const onClick = useMemo(() => {
  //   return () => {
  //     console.log("Click");
  //   };
  // }, []);
  // const onClick = useCallback(() => {
  //   console.log("Click");
  // }, []);
  return (
    <>
      <span>{count}</span>
      <input
        type="button"
        onClick={() => setCount(count + 1)}
        value="修改count"
      />
      <Counter onClick={onClick}/>
    </>
  );
}
```