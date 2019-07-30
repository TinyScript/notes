# Suspense组件
Suspense有点像javascript中的`async/await`，做组件异步加载，但是没写好容易导致浏览器死循环，要小心使用。

`Suspense组件`要配合`Lazy组件`使用，因为React组件没办法用`Promise`，也无法像Vue用`import()`懒加载，所以只能通过React.Lazy进行`import()`。

Suspense组件提供一个fallback属性，可以接收任意html或者loading组件。


Suspense的执行原理：

1、Suspense的原理是内部会一直`轮询`一个Promise对象，并且判断Promise.resolve的值得是否与上一次的`then接收的resolve值`相等。

2、如果`相等`的情况下，会将promise实例`throw出去`，继续下一次的轮询，如果这里没有throw那就要造成死循环了。

3、如果碰到`不相等`的情况下，那就说明异步组件已经完成，返回异步组件。


因为这样的操作有点傻，所以React向开发者提供了`react-hooks-fetch`进行调用，这样就无需考虑太多，通过`ReactHooksFetch.useFetch`请求回来的data与error进行渲染，有data渲染data，有error渲染error，当然也要做好无data时的处理。


如果觉得`react-hooks-fetch`不够用，可以试试`react-hooks-async`，它甚至还可以与axios混合使用，解决请求头配置等问题。

### index.jsx
```javascript 
//新增了render 新的返回类型：fragments 和 strings
import React, { Suspense, lazy } from "react";
import "./suspense.css";
import { useFetch } from "react-hooks-fetch";
// console.log("异步加载数据", useFetch);
//动态加载组件
const LazyComp = lazy(() => import("./lazy"));

function fetchApi() {
  const promise = new Promise(resolve => {
    setTimeout(() => {
      resolve("Data resolved");
    }, 3000);
  });
  return promise;
}
//创建Fetcher
var cached = {};
const createFetcher = promiseTask => {
  let ref = cached;
  return () => {
    const task = promiseTask();
    task.then(res => {
      ref = res;
    });
    console.log("🌲--ref", ref);
    console.log("🌺--cached", cached);
    if (ref === cached) {
      throw task;
    }
    //得到结果输出
    console.log("🍎", ref);
    return ref;
  };
};
const requestData = createFetcher(fetchApi);
function SuspenseComp() {
  // const { error, data } = useFetch("a.php");
  // // console.log("数据📚",data)
  // if (error) return <span>出错了/(ㄒoㄒ)/~~</span>;
  // if (!data) return null;
  // return <span>RemoteData:{data.title}</span>;
  const data = requestData();
  return <p className="text-warning">{data}</p>;
}

export default () => (
  <Suspense
    fallback={
      <div className="text-danger">
        loading<i />
      </div>
    }
  >
    <SuspenseComp />
    <LazyComp />
  </Suspense>
);
```

### lazy.jsx
```javascript
import React from 'react'
export default () => <p className="text-success">我是Lazy Comp</p>
```