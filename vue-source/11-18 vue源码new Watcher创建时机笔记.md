### 0x00 Watcher的创建时机
在宏观的角度来讲，vue有三个情况会创建watcher：
1. 挂载component时
2. 初始化computed时
3. 调用$watch方法时

### 0x01 挂载component
`component`在Vue._init函数执行阶段初始化，在挂载之前会进行new Watcher。

也就是在`vm.$mount -> mountComponent`这里开始进行，并创建了一个watcher实例。

```javascript 
new Watcher(vm, updateComponent, noop, {
  before () {
    if (vm._isMounted && !vm._isDestroyed) {
      callHook(vm, 'beforeUpdate')
    }
  }
}, true /* isRenderWatcher */)
```

然后在new Watcher时会把实例给到vm._watchers进行管理，最后调用watcher.get时会调用updateComponent函数。

执行路径：
`new Vue` -> `this._init` -> `vm.$mount` -> `mountComponent` -> `new Watcher`

### 0x02 初始化computed
`computed`在initState时期初始化，然后执行`initComputed`，在initComputed里面new Watcher。

在这个过程中会遍历computed所有的key，并进行初始化watcher，此时有两个地方会保存这些watcher。

1. vm.computedWatchers: ==Object==
2. vm._watchers: ==Array==

```
// 1、Object
const watchers = vm._computedWatchers = Object.create(null)

for(const key in computed) {
    watchers[key] = new Watcher(
      vm,
      getter || noop,
      noop,
      computedWatcherOptions
    )
}

// 2、Array
class Watcher {
  constructor(
    vm, 
    expOrFn, 
    cb, 
    options, 
    isRenderWatcher
  ) {
    vm._watchers.push(this)
  }
}

```

执行顺序：

`new Vue` -> `this._init` -> `initState` -> `initComputed` -> `new Watcher`

### 0x03 调用$watch方法
调用`vm.$watch`方法时也会new Watcher，并且返回这个watcher实例本身拆解监听的方法。

传入首个参数就是触发回调用的参数。

```javascript
Vue.prototype.$watch = function(expOrFn, cb, options) {
  const watcher = new Watcher(vm, expOrFn, cb, options)   
}
```

### 0x04 总结
watcher执行的源头目前发现的有三个地方：
1. 监听component
2. 监听computed
3. 监听$watch绑定的事件

下次找找依赖收集是怎么一回事。