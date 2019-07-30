# Memo组件
Memo组件相当于记忆组件，它适用于函数组件，不适用于类组件，目的是让一个`函数组件`变成一个`纯组件`，作用与`React.PureComponent`非常相似。

第二个参数areEqual的函数参数值与shouldComponentUpdate的返回值相反


```javascript
//React.memo() 是高阶函数能将函数组件转换成类似于React.PureComponent组件
import React from "react";
function Child({seconds}){
    console.log('I am rendering');
    return (
        <div>I am update every {seconds} seconds</div>
    )
};
export default React.memo(Child)
```
