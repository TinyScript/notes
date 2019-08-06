# redux用法
> 以下记录是阅读`React Redux 7.1`文档所记录。

一开始看redux就有种与vuex不太一样的地方，vuex会将state、getters、actions、mutations集中处理。

而redux则是需要用于手动维护进行连接，比如需要React Redux让redux与react进行连接，听起来就跟后台的数据库连接一样。

首先redux需要在外层使用`Provider`组件进行包裹，并将`store`传入Provider标签的store属性中。

```javascript
import { Provider } from 'react-redux'
import store from './store/index'
import App from './App'

<Provider store={store}>
  <App />
</Provider>
```

然后再通过`connect()`方法进行`redux数据`与`react组件`连接。connect接收两个参数：`state`与`dispatch`，传给react的组件。对于文档中的参数名，顾名思义，最后会在react中转换成props调用。

```javascript
import { connect } from 'react-redux'
import { increment, decrement, reset } from './actionCreators'

class Counter extends React.component {
  getState() {
    return this.props.counter
  }
  callDispatch() {
    this.props.increment();
    this.props.decrement();
    this.props.reset();
  }
} 

const mapStateToProps = (state) => {
  return {
    counter: state.counter
  }
}
const mapDispatchToProps = { increment, decrement, reset }

export default connect(
  mapStateToProps,
  mapDispatchToProps
)(Counter)

```