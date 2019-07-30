# hoc高阶组件
有了函数式组件，高阶组件就衍生出来了。高阶函数是传入函数返回一个新函数，高阶组件则是传入组件返回一个新的组件，讲究的就是一个纯字。


以下面的代码为例，如果没有高阶组件，Welcome与GoodBye组件可能要重复定义state，并且还要重复写在生命周期中获取localStorage的值，这样就失去组件化的优势了。


然而我们可以通过高阶组件去避免这种情况的发生：

1、通过定义一个函数用于接收一个将要做欢迎动作的`组件A（Welcome）`。

2、该函数拿到组件A后，通过一系列的操作返回一个`新组件（NewComponent）`，以上的两个步骤就是核心思想，下面继续细说过程。

3、要做欢迎动作的`组件A`不知道给谁欢迎，所以这时就是新组件要做功的时候了。

4、新组件在`componentWillMount`获取了localStorage的数据并setState到了username中。

5、再通过`render`将username给到了组件A。

6、此时`组件A`只需要对新组件给到的`props.username`打招呼即可。

7、这样就可以做到一个很好的解耦，也保持了组件之间的纯度，符合函数式编程的要求。

```javascript
//高阶组件就是一个没有副作用的纯函数。
import React, { Component } from "react";
const wrapWithUsername = WrappedComponent => {
  class NewComponent extends Component {
    constructor() {
      super();
      this.state = {
        username: ""
      };
    }
    componentWillMount() {
      let username = localStorage.getItem("username");
      this.setState({
        username: username
      });
    }

    render() {
      return <WrappedComponent username={this.state.username} />;
    }
  }

  return NewComponent;
};
class Welcome extends Component {
  render() {
    return <div className="text-warning">welcome {this.props.username}</div>;
  }
}
//升级高阶组件
Welcome = wrapWithUsername(Welcome);

class Goodbye extends Component {
  render() {
    return <div className="text-info">goodbye {this.props.username}</div>;
  }
}
//升级高阶组件
Goodbye = wrapWithUsername(Goodbye);
class Greeting extends Component {
  render() {
    return (
      <>
        <Welcome /> <Goodbye />
      </>
    );
  }
}
export default Greeting;
```