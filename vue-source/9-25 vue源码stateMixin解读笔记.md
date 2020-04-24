#### stateMixin
`stateMixin`在Vue.prototype双向绑定了`$data`和`$props`属性，并定义了`$set`与`$delete`方法，和`$watch`方法。

```javascript
export function stateMixin (Vue: Class<Component>) {
  const dataDef = {}
  dataDef.get = function () { return this._data }
  const propsDef = {}
  propsDef.get = function () { return this._props }

  Object.defineProperty(Vue.prototype, '$data', dataDef);
  Object.defineProperty(Vue.prototype, '$props', propsDef);

  Vue.prototype.$set = set;
  
  Vue.prototype.$delete = del;
  
  Vue.prototype.$watch = watch;
}
```

stateMixin主要混入了如下属性：
- $data
- $props
- $set
- $delete
- $watch

#### observer/index.js -- function set
`set`方法就是我们常用的$set方法，强行绑定一个新属性到vue的data中，无论如何都会给你return你要设置的val，分几种情况：
1. 数组直接set
2. 原型链上的key直接set
3. 如果target.__ob__没有值，直接set
4. 如果是vue对象或者有`target.__ob__.vmCount`则报警告没有set方法

```javascript
export function set (target: Array<any> | Object, key: any, val: any): any {
  // 数组
  if (Array.isArray(target) && isValidArrayIndex(key)) {
    target.length = Math.max(target.length, key)
    target.splice(key, 1, val)
    return val
  }
  // key在target中，且不是Object.prototype的属性，直接赋值
  if (key in target && !(key in Object.prototype)) {
    target[key] = val
    return val
  }
  // 根据target.__ob__判断，没有ob，则直接给target赋值
  const ob = (target: any).__ob__
  if (!ob) {
    target[key] = val
    return val
  }
  // 有ob，则给ob.value的key与val赋值
  defineReactive(ob.value, key, val)
  // 告诉依赖收集者给订阅者发布通知
  ob.dep.notify()
  return val
}
```

`del`方法与`set`几乎一致，不同的则是删除的操作

`watch`方法接收了3个参数`keyOrFn`、`cb`、`options`，针对cb是对象时会继续`createWatcher`，继续走$watch方法监听。然后new Watcher，并拿到watcher实例。如果设置了`immediate`字段，则会立刻执行cb的。然后返回一个解除watcher监听的函数。
```javascript
Vue.prototype.$watch = function (
  expOrFn: string | Function,
  cb: any,
  options?: Object
): Function {
  const vm: Component = this
  if (isPlainObject(cb)) {
    return createWatcher(vm, expOrFn, cb, options)
  }
  options = options || {}
  options.user = true
  const watcher = new Watcher(vm, expOrFn, cb, options)
  if (options.immediate) {
    try {
      cb.call(vm, watcher.value)
    } catch (error) {
      handleError(error, vm, `callback for immediate watcher "${watcher.expression}"`)
    }
  }
  return function unwatchFn () {
    watcher.teardown()
  }
}
```

#### observer/watcher.js -- function createWatcher

`createWatcher`则是对需要做deep之类的深度监听做的操作。看到handler就很熟悉了，函数最终还是会跑一遍$watch

```javascript
function createWatcher (
  vm: Component,
  expOrFn: string | Function,
  handler: any,
  options?: Object
) {
  if (isPlainObject(handler)) {
    options = handler
    handler = handler.handler
  }
  if (typeof handler === 'string') {
    handler = vm[handler]
  }
  return vm.$watch(expOrFn, handler, options)
}
```

#### observer/watcher.js -- function Watcher.constructor

`Watcher.constructor`主要是调用`this.get`方法给到`this.value`，并对watch非函数的对象进行处理，对`this.getter`变量进行处理。
```javascript
constructor(
    vm: Component,
    expOrFn: string | Function,
    cb: Function,
    options?: ?Object,
    isRenderWatcher?: boolean
) {
    this.vm = vm
    // ...
    vm._watchers.push(this)
    // options
    if (options) {
      this.deep = !!options.deep
      this.user = !!options.user
      this.lazy = !!options.lazy
      this.sync = !!options.sync
      this.before = options.before
    } else {
      this.deep = this.user = this.lazy = this.sync = false
    }
    // ...
    this.id = ++uid // uid for batching
    // ...
    // parse expression for getter
    if (typeof expOrFn === 'function') {
      this.getter = expOrFn
    } else {
      this.getter = parsePath(expOrFn)
      if (!this.getter) {
        this.getter = noop
        // 报警告
    }
    
    this.value = this.lazy
      ? undefined
      : this.get()
}
```

#### observer/watcher.js -- function Watcher.get
`Watcher.get`方法开头和结尾都会针对Dep进行`pushTarget`与`popTarget`。目的是为了通过`this.getter.call(vm, vm)`拿到value值。如果存在deep属性则将value给到`traverse`函数进行深度监听。

```javascript
get () {
  pushTarget(this)
  let value
  const vm = this.vm
  try {
    value = this.getter.call(vm, vm)
  } catch (e) {
    if (this.user) {
      handleError(e, vm, `getter for watcher "${this.expression}"`)
    } else {
      throw e
    }
  } finally {
    // "touch" every property so they are all tracked as
    // dependencies for deep watching
    if (this.deep) {
      // 深度监听
      traverse(value)
    }
    popTarget()
    this.cleanupDeps()
  }
  return value
}
```

`traverse`传递val与seenObject给`_traverse`函数进行调用，内部则对`val.__ob__.dep.id`进行收集。然后对val的类型进行区分处理，数组则遍历traverse数组，对象则遍历traverse对象。
```javascript
export function traverse (val: any) {
  _traverse(val, seenObjects)
  seenObjects.clear()
}

function _traverse (val: any, seen: SimpleSet) {
  let i, keys
  const isA = Array.isArray(val)
  if ((!isA && !isObject(val)) || Object.isFrozen(val) || val instanceof VNode) {
    return
  }
  
  // 收集依赖id
  if (val.__ob__) {
    const depId = val.__ob__.dep.id
    if (seen.has(depId)) {
      return
    }
    seen.add(depId)
  }
  
  // 区分处理
  if (isA) {
    i = val.length
    while (i--) _traverse(val[i], seen)
  } else {
    keys = Object.keys(val)
    i = keys.length
    while (i--) _traverse(val[keys[i]], seen)
  }
}
```