# State 
> type State = any;

*State*也叫state tree，与vuex中的*State*相似，由`store`管理，通过`getState()`方法获取，它表示了Redux的全部状态。

顶层的*State*可以是对象，或者像Map那样的`key-value对`集合，也可以是任意的数据类型。

但是！你应该尽可能的让*State*是可以被序列化的数据结构，而且不要把什么数据都放进去，导致无法轻松的把*State*转换成JSON结构。

重点：
- `state`由`store`管理。
- 通过`getState()`方法获取state值。
- `顶层state`可以是key-value的引用类型，或任意的数据类型。
- 尽可能保证state是一个可以被`序列化`的结构。