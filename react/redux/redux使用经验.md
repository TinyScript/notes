# redux使用经验
> 主要是讲解最近学react用到redux的一个理解

**React**是一个前端模板，有组件的概念，因为组件的存在，受业务的影响肯定会出现`组件之间的通信`这种需求。组件之间可以通过props进行父子组件的通信，流程与vue几乎一样，这算是一个老生常谈的问题了，弊端肯定就是组件跨度太大时不好维护，数据也不好追踪，所以要用到redux。

**Redux**则是一个数据仓库，用来管理公共数据的一块内存，vuex的思想就是延伸自这里。redux有几个基本词store，state，action，reducer，dispatch这里就不多做介绍。

**React Redux**则是将redux与react进行一个连接使用，它会提供`Provider`组件用来接收store，并包裹整个App组件，方便对所有子组件进行数据派送。并提供`connect`的方法，将`state数据`与`action方法`注入到`React.Component`中，对redux的数据进行操作。

**Provider组件**会接收一个store，包裹App的最外层组件。

**connect**会提供两个参数，`mapStateToProps`与`mapActionToProps`，顾名思义，这样react组件就可以通过props进行调用redux注入的state与action。

>mapStateToProps接收一个函数，并提供state参数由你拆解，并返回一个新的对象。

>mapActionToProps接收一个action函数的对象。
- type mapStateToProps = (state) => {}
- type mapActionToProps = {}