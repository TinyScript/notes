# useRef
`forwardRef`与`useImperativeHandle`钩子是成对使用的。

如果你在父组件中使用了`const myRef = useRef`，然后将`myRef`传入到使用forwardRef创建的子组件函数中，再使用`useImperativeHandle`对`forwardRef`传入的第二个参数ref进行绑定方法。

然后父组件就可以通过`myRef.current`进行访问`useImperativeHandle`定义的内容了。

这样封装组件的时候，就可以提供一个钩子给予使用者，他们可以拿到内部的组件并对其进行自定义的配置，提高组件封装的灵活性。

```javascript
import React, {
  forwardRef,
  useRef,
  useEffect,
  useImperativeHandle
} from "react";

const ContextComp = forwardRef((props, ref) => {
  useImperativeHandle(ref, () => ({
    method() {
      console.log("ref方法执行");
    }
  }));

  return <p ref={ref}>子组件</p>;
});

export default function App() {
  const myRef = useRef();
  useEffect(() => {
    console.log("component update");
    myRef.current.method();
    return () => {
      console.log("unbind");
    };
  }, []);
  return (
    <>
      <ContextComp ref={myRef} />
    </>
  );
}
```