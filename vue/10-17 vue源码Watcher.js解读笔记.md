### 0x00 class -- Watcher

`Watcher类`其实就是承担了订阅发布模式中的`Subscribe`的角色，这个在`Dep.js`文件已经分析说明过了。

所以`Watcher类`的实例肯定就会有一个`update`的方法用于之后notify。

```javascript
class Watcher {
    constructor() {}
    update() {}
}
```

### 0x01 哪里用了new Watcher

先看哪里调用了new Watcher：
1. lifecycle.js中的mountComponent函数末尾

```javascript
export function mountComponent (
  vm: Component,
  el: ?Element,
  hydrating?: boolean
): Component {

  /** 
    * something code...
    */

  new Watcher(vm, updateComponent, noop, {
    before () {
      if (vm._isMounted && !vm._isDestroyed) {
        callHook(vm, 'beforeUpdate')
      }
    }
  }, true /* isRenderWatcher */)

  /** 
    * something code...
    */
  
  return vm
}
```
2. 在state.js中的initComputed函数，遍历所有computed会实例化Watcher

```javascript
function initComputed (vm: Component, computed: Object) {
  const watchers = vm._computedWatchers = Object.create(null)
  const isSSR = isServerRendering()
  for (const key in computed) {
    const userDef = computed[key]
    const getter = typeof userDef === 'function' ? userDef : userDef.get
    
    if (!isSSR) {
      // create internal watcher for the computed property.
      watchers[key] = new Watcher(
        vm,
        getter || noop,
        noop,
        computedWatcherOptions
      )
    }
    
    /** 
      * something code...
      */
  
  }
}
```
3. state.js的stateMixin的Vue.prototype.$watch中

```javascript
export function stateMixin (Vue: Class<Component>) {
  Vue.prototype.$watch = function (
    expOrFn: string | Function,
    cb: any,
    options?: Object
  ): Function {
    // something code...
    const watcher = new Watcher(vm, expOrFn, cb, options)
    // something code...    
  }
}
```

总结一下上面的3个新建实例化Watcher时的场景：
1. 更新组件
2. 更新computed
3. 更新state（data）

### 0x02 function constructor

现在我们看一下new Watcher时都干了什么。

首先来到`constructor`函数，接收了5个参数：
- vm -- vue实例
- expOrFn -- 表达式或者函数
- cb -- 回调函数
- options -- 配置信息
- isRenderWatcher -- 是否在vm渲染当前的watcher

```javascript
export default class Watcher {
  constructor(
    vm: Component,
    expOrFn: string | Function,
    cb: Function,
    options: Object,
    isRenderWatcher: boolean
  ) {
      // something code...
  }
}
```

每一次实例化Watcher都会给vm的`_watchers`push自己。如果需要渲染单独的自己，则通过传入参数`isRenderWatcher`来处理

```javascript
// watcher是单数
if(isRenderWatcher) {
  vm._watcher = this 
}
// watchers是复数，也就是一个数组
vm._watchers.push(this)
```

判断传入参数`options`，如果有传入则给对应的成员变量进行处理。如果没有传入`options`，四个布尔成员变量则为false
```javascript
if(options) {
  this.deep = !!options.deep
  this.user = !!options.user
  this.lazy = !!options.lazy
  this.sync = !!options.sync
  this.before = options.before
} else {
  this.deep = this.user = this.lazy = this.sync = false
}
```

之后就是初始化各种watcher用到的成员变量

```javascript
this.cb = cb
this.id = ++uid
this.active = true
this.dirty = this.lazy
this.deps = []
this.newDep = []
this.depIds = new Set()
this.newDepIds = new Set()
```

通过判断expOrFn参数的类型，对`this.getters`进行处理，如果是函数则直接给getters，否则通过`parsePath`函数解析路径，再由后续传入的对象进行获取。
```javascript
if(typeof expOrFn === 'function') {
  this.getters = expOrFn
} else {
    this.getters = parsePath(expOrFn)
    if(!this.getters) {
      this.getters = noop
      warn()
    }
}
```

最后处理`this.value`成员变量，如果options有传lazy，则给this.value一个undefined，否则调用`this.get()`方法拿到值给`this.value`
```javascript
this.value = this.lazy ? undefined : this.get()
```

### 0x03 function get

接着上面顺便说一下`this.get`方法。调用该方法的时候会先告诉Dep这个订阅者，目前谁正在处理谁，通过`pushTarget`与`popTarget`。最后返回需要被处理的value变量，先来看看粗略版。
```javascript
get () {
  pushTarget(this) // 告诉Dep正在处理谁
  let value
  const vm = this.vm
  
  try { /* something code... */ }
  catch(e) { /* something code... */ }
  finally {
    popTarget() // 处理完成后告诉Dep要离开了
  }
  
  return value
}
```

理想状态下就是直接调用getters然后返回value。如果`options.deep`有值则对value进行递归访问，触发每个属性的`get函数`，然后清除`clearupDeps`依赖
```javascript
try {
  value = this.getters.call(vm, vm)
} catch(e) {
  throw e
} finally {
  if(this.deep) {
    traverse(value)
  }
  popTarget()
  this.cleanupDeps()
}

return value
```

### 0x04 observer/traverse.js -- function traverse
顺着刚刚的finally部分执行的函数，到了`traverse.js`。

1. 这个文件定义了一个`seenObjects`的set数据结构，用来收集所有`vm对象`，准确的说是有`vm.__ob__`的对象。
2. 定义了一个`traverse函数`接收一个`val`参数，里面执行了`_traverse(val, seenObejcts)`函数。
3. 最后将seenObjects清理干净。

```javascript
const seenObjects = new Set()

export function traverse (val: any) {
  _traverse(val, seenObjects)
  seenObjects.clear()
}
```

`_traverse`函数要做的就是递归遍历所有的val，触发它们`defineProperty`的`get方法`。
```javascript
// 核心代码
while (i--) _traverse(val[i], seen)
```

首先会针对val进行类型的判断，不满足以下类型的全部停止递归。
- 非数组
- 非对象
- 冻结对象
- 或者不属于VNode的派生实例
```javascript
const isA = Array.isArray(val)
// 是否数组，是否对象 || 是否冻结对象 || 是否虚拟节点的派生实例
if (
    (!isA && !isObject(val)) || 
    Object.isFrozen(val) || 
    val instanceof VNode
) {
  return
}
```

接下来判断val如果存在`__ob__`属性，则取到`__ob__.dep.id`并将id存入seenObjects，以防下次遇到相同的还继续往下走。
```javascript
if (val.__ob__) {
  const depId = val.__ob__.dep.id
  if (seen.has(depId)) {
    return
  }
  seen.add(depId)
}
```

最后针对val进行对象和数组的区分处理，其实都一样，就是为了遍历数组和对象，然后对每个子val进行递归访问。
```javascript
if (isA) {
  i = val.length
  while (i--) _traverse(val[i], seen)
} else {
  keys = Object.keys(val)
  i = keys.length
  while (i--) _traverse(val[keys[i]], seen)
}
```

全量代码
```javascript
function _traverse (val: any, seen: SimpleSet) {
  let i, keys
  const isA = Array.isArray(val)
  // 是否数组，是否对象 || 是否冻结对象 || 是否虚拟节点的派生实例
  if ((!isA && !isObject(val)) || Object.isFrozen(val) || val instanceof VNode) {
    return
  }
  // val存在__ob__时说明是vm对象，拿到它的依赖id。
  // 如果已经有依赖id则不做处理，否则add依赖id
  if (val.__ob__) {
    const depId = val.__ob__.dep.id
    if (seen.has(depId)) {
      return
    }
    seen.add(depId)
  }
  // 如果它是一个数组，则对数组进行递归调用_traverse
  // 否则用对象的方式进行递归调用
  // seenObjects会在当次调用一直记录，相当于路径，递归调用完毕时会clear所有记录
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

### 0x05 function cleanupDeps

`cleanupDeps`是为了清空deps列表的订阅，以及各种成员变量。

首先遍历deps，如果在newDepIds中不存在该dep[i].id，则取消订阅。

接着就是将`newDepIds给到depIds`、`newDeps给到deps`。再清理this.newDepIds，this.newDeps两个列表。
```javascript
cleanupDeps () {
  let i = this.deps.length
  while (i--) {
    const dep = this.deps[i]
    if (!this.newDepIds.has(dep.id)) {
      dep.removeSub(this)
    }
  }
  let tmp = this.depIds
  this.depIds = this.newDepIds
  this.newDepIds = tmp
  this.newDepIds.clear()
  tmp = this.deps
  this.deps = this.newDeps
  this.newDeps = tmp
  this.newDeps.length = 0
}
```

### 0x06 function update
构造器及相关用到函数看完了，现在看看订阅发布模式中subscribe的核心方法--`update`。

1. 如果options传入`lazy`，则给dirty标记为true，目前暂时还不清楚是干什么用的，有的人说是computed延迟处理用。
2. 如果options传入`sync`，则调用`this.run()函数`，当然在下面的queueWatcher函数也会调用this.run函数
3. 否则调用`queueWatcher(this)`，默认的Watcher是扔进队列等待处理。

```javascript
update () {
  if (this.lazy) {
    // computed特有的处理
    this.dirty = true
  } else if (this.sync) {
    this.run()
  } else {
    // 默认会执行的队列
    queueWatcher(this)
  }
}
```

### 0x07 function run
`run`函数除了在sync同步的情况下，还要在当前watcher处于active状态才能做事。

```javascript
run() {
  if(this.active) {
      // do something...
  }
}
```

最终run函数的处理跟传入的watch很像，处理`this.value`与`this.oldValue`，然后将他们给到`this.cb函数`。

给this.cb函数之前，还做了三层判断
- value与this.value不等时
- value是object时
- this.deep存在时，代表需要深度处理
```javascript
const value = this.get()
if(
  value !== this.value ||
  isObject(value) ||
  this.deep
)
const oldValue = this.value
this.value = value

this.cb.call(this.vm, value, oldValue)
```

### 0x08 scheduler.js -- function queueWatcher
`queueWatcher`专门用来处理watcher队列，函数外部定义了一个`哨兵变量has`，专门用于记录watcher id用。最终目的是为了在下一次数据更新修改页面后，调用`flushSchedulerQueue`清洗队列，清洗日程队列也就是处理日程队列的意思。

```javascript
queueWatcher(watcher: Watcher) {
  const id = watcher.id
  if(has[id] == null) {
    has[id] = true
    /**
     * do something...
     **/
    nextTick(flushSchedulerQueue)
  }
}
```

函数内部有两个分支判断条件，是否清洗中`flushing`，如果不在清洗中的状态，给`queue队列`推送一个watcher
```javascript
if(!flushing) {
  queue.push(watcher)
} else {
  // do something...
}
```

在清洗队列状态中，会根据当前队列清洗的位置index对比：`i > index`，比上了再让当前队列中最后watcher的id与插入的watcher的id进行对比：`queue[i].id > watcher.id`。

完了以后给队列queue进行watcher插入。
```javascript
if(!flushing) {
  // do something...
} else {
  let i = queue.length - 1 // 初始化i代表队列最后一位索引
  // 队列长度大于当前正在遍历的索引
  // 队列当前的watcher.id大于即将被插入的watcher.id
  while(i > index && queue[i].id > watcher.id) {
    i-- // 插入位子再往前走一个
  }
  // 在确定索引位置后添加一位watcher
  queue.splice(i+1, 0, watcher)
}
```

最后判断`wating`哨兵变量，如果非等待状态就开始处理。wating就是起到节流作用。然后在下一次视图修改后触发队列清洗函数`flushSchedulerQueue`。

```javascript
if(!wating) {
  wating = true
  nextTick(this.flushSchedulerQueue)
}
```

queueWatcher全量代码：
```javascript
export function queueWatcher (watcher: Watcher) {
  const id = watcher.id
  if (has[id] == null) {
    has[id] = true
    if (!flushing) {
      queue.push(watcher)
    } else {
      let i = queue.length - 1
      while (i > index && queue[i].id > watcher.id) {
        i--
      }
      queue.splice(i + 1, 0, watcher)
    }
    // queue the flush
    if (!waiting) {
      waiting = true

      if (process.env.NODE_ENV !== 'production' && !config.async) {
        flushSchedulerQueue()
        return
      }
      nextTick(flushSchedulerQueue)
    }
  }
}
```

### 0x09 function flushSchedulerQueue
这个函数就是刚刚一直说的，清洗日程队列，也就是传说中的`调度算法`。

来吧，从头开始读，开始阶段先拿现阶段的timestamp，然后标记清洗阶段，再对队列中每个watcher的id进行排序。并且定义了`watcher`和`id`两个变量。
```javascript
export function flushSchedulerQueue() {
  const timestamp = getNow()
  flushing = true
  let watcher, id
  queue.sort((a, b) => a.id - b.id)
  
  /**
   * something code...
   **/

}
```

然后就是对队列queue进行处理，处理完则对watcher的`外部变量has`进行对应id键的值进行置空，并执行刚刚看说到过的函数`watcher.run`
```javascript
for(index = 0; index < queue.length; index++) {
  watcher = queue[index]
  id = watcher.id
  has[id] = null
  watcher.run()
}
```

最后对`activatedQueue`和`updateQueue`两个队进行缓存，再调用清空队列函数，最后调用两个缓存队列的钩子，如果存在devtools调用`devtools.emit`送一个flush的事件。
```javascript
const activatedQueue = activateQueue.slice()
const updateQueue = queue.slice()

resetSchedulerState()

callActivatedQueueHook(activatedQueue)
callUpdateQueueHook(updateQueue)
if(devtools && config.devtools) {
  devtools.emit('flush')
}
```

flushSchedluerQueue全量代码：
```javascript
function flushSchedulerQueue () {
  currentFlushTimestamp = getNow()
  flushing = true
  let watcher, id

  queue.sort((a, b) => a.id - b.id)

  for (index = 0; index < queue.length; index++) {
    watcher = queue[index]
    if (watcher.before) {
      watcher.before()
    }
    id = watcher.id
    has[id] = null
    watcher.run()
    
    if (process.env.NODE_ENV !== 'production' && has[id] != null) {
      circular[id] = (circular[id] || 0) + 1
      if (circular[id] > MAX_UPDATE_COUNT) {
        warn(
          'You may have an infinite update loop ' + (
            watcher.user
              ? `in watcher with expression "${watcher.expression}"`
              : `in a component render function.`
          ),
          watcher.vm
        )
        break
      }
    }
  }

  const activatedQueue = activatedChildren.slice()
  const updatedQueue = queue.slice()

  resetSchedulerState()

  callActivatedHooks(activatedQueue)
  callUpdatedHooks(updatedQueue)
  
  if (devtools && config.devtools) {
    devtools.emit('flush')
  }
}
```

总结一下flushSchedulerState作用。
1. 拿到timestamp给到哨兵变量，currentFlushTimestamp。
2. 对当前的queue按`watcher.id`进行排序。
3. 遍历`const watcher = queue[i]`，再对`watcher.before`、`watcher.run`进行处理，并置空`has[watcher.id]`。
4. 复制一份activatedChildren到activatedQueue备用。
5. 复制一份queue到updateQueue备用。
6. 清空日程状态，resetSchedulerState。
7. 调用activated和update的钩子：`callActivatedHooks(activatedQueue)`、`callUpdateHooks(updateQueue)`。
8. 对devtools进行处理，如果存在devtools&&devtools.config，发射一个事件`devtools.emit`

### 0x0A function addDeps
`addDeps`将自己添加到Dep类的list中。这个方法比较简单，接收一个dep，只有几行代码，两个判断。

```javascript
addDep(dep: Dep) {
  // do something...    
}
```

首先判断传入参数的id`dep.id`是否在`this.newDepIds`列表中。如果不在，则给`this.newDepIds`Set集合添加该id，给`this.newDeps`数组推该dep实例。

```javascript
const id = dep.id
if(!this.newDepIds.has(id)) {
  // id用Set保存
  this.newDepIds.add(id)
  // dep实例用数组保存
  this.newDeps.push(dep)
  // do something...
}
```

然后再判断当前`this.depIds`Set集合是否存在该id，没有的话则给dep添加当前watcher对象。

注意：是判断`Watcher类`的成员变量depIds集合是否存在id，再用`Dep类`的addSub方法，给`dep实例`的subs列表添加`watcher实例`

```javascript
if(!this.depIds.has(id)) {
  dep.addSub(this)
}
```

addDeps全量代码：
```javascript
/**
 * Add a dependency to this directive.
 * 通过这个指令添加依赖
 */
addDep (dep: Dep) {
  const id = dep.id
  if (!this.newDepIds.has(id)) {
    this.newDepIds.add(id)
    this.newDeps.push(dep)
    if (!this.depIds.has(id)) {
      dep.addSub(this)
    }
  }
}
```

### 0x0B function evaluate
这个方法比较简单，只针对`this.value`进行`this.get()`赋值，这个看起来跟constructor的函数基本一样。然后再对`this.dirty`赋值为false。

这里就跟`constructor`最后一步对上了，如果`new Watcher`时，`options.lazy`为true时，this.value为undefined。

然后在`state.js`中的`createComputedGetter`调用`watcher.evaluate`函数。也就是说computed是带有lazy属性的，会等component将所有属性初始化好了，再处理computed属性。

函数执行路径：`initState` --> `initComputed` --> `definedComputed` --> `createComputedGetter` --> `computedGetter` --> `watcher?.dirty?.evaluate()`

```javascript
evaluate() {
  this.value = this.get()
  this.dirty = false
}
```

总结：虽然代码量不多，但是在Vue的`computed`起到承上启下的作用。

### 0x0C function depend
在这里先记录一下花了半小时才搞明白的问题概要：`watcher.depend`与`dep.depend`方法不一样。

`watcher.depend`是调用watcher中`this.deps列表`所有的deps[i].depend方法。
```javascript
class Watcher {
  // 给watcher中的依赖列表添加依赖项。
  addDep(dep: Dep) {
    const id = dep.id
    if(!this.newDepIds.has(id)) {
      this.newDepIds.add(id)
      this.newDeps.push(dep)
      if(!this.depIds.has(id)) {
        dep.addSub(this)
      }
    }
  }
  
  depend () {
    let i = this.deps.length
    while (i--) {
      this.deps[i].depend()
    }
  }
}
```
`dep.depend`则会访问Dep的target静态变量，如果Dep.target存在则给Dep.target当前存的watcher实例添加依赖，也就是watcher.addDep：`Dep.target.addDep(this)`
```javascript
class Dep {
  addSub(sub: Watcher) {
    this.subs.push(sub)
  }
  
  depend () {
    if (Dep.target) {
      Dep.target.addDep(this)
    }
  }
}
```

执行路径：`watcher.depend()` --> `dep.depend()` --> `watcher.addDep(dep)` --> `dep.addSub(watcher)`

在`state.js`的`computedGetter`函数会根据Dep.target是否存在给当前computed的watcher执行`watcher.depend()`

而在`observer.js`中的`defineReactive`函数，会新建一个Dep实例到dep变量，然后将所有对象中defineProperty的get进行拦截，只要触发了get就会判断`Dep.target`再调用`dep.depend`进行依赖。

`watcher.depend` + `dep.depend`总结：
1. 通过`watcher.depend`触发`dep.depend`，或者直接触发`dep.depend`。
2. 判断Dep的静态变量target是否有存`watcher`，有则调用`Dep.target.addDep(this)`，让`watcher实例`与`dep实例`产生联系。通过`pushTarget`和`popTarget`方法控制的Dep.target。
3. 如果watcher中的`newDepIds`和`depIds`都没有新增的dep.id记录，则说明watcher本身没有与dep产生联系，调用`this.dep.addSub(this)`进行连接。


### 0x0C function teardown
`teardown`拆卸函数，将自己与所有的dependencies列表进行拆除，注释就是这么写的。

首先如果自己的`active`标记是处于true--激活的状态时，才会执行拆卸动作。
```javascript
teardown() {
  if(this.active) {
    // do something...
  }
}
```

如果当前观察者的组件处于销毁的状态时，则跳过删除自己这一动作，不用多此一举。
```javascript
if(!this.vm._isBeingDestroyed) {
  remove(this.vm.watchers, this)
}
```

再给与watcher实例发生联系的dep实例进行拆卸。
```javascript
let i = this.deps.length
while(i--) {
  this.deps[i].removeSub(this)
}
```

当把自己从所有有关系的deps拆卸完成时，顺便把active关闭。
```javascript
this.active = false
```

teardown的全量代码：
```javascript
/**
 * Remove self from all dependencies' subscriber list.
 * 从全部有依赖关系的订阅者列表删除watcher自己。
 */
teardown () {
  if (this.active) {
    if (!this.vm._isBeingDestroyed) {
      remove(this.vm._watchers, this)
    }
    let i = this.deps.length
    while (i--) {
      this.deps[i].removeSub(this)
    }
    this.active = false
  }
}
```

总结：
#### 初始化时
在data、computed、组件会有watcher进行观察。

在watcher初始化就会执行一次getters，也就是传入的expOrFn函数。

如果getters得到的value是一个对象，则会对这个对象进行递归遍历，触发这个对象所有的get拦截器。

#### update 
update有三个分支：lazy、sync、normal。

lazy是computed用的，sync没找到，初步猜测是用于组件的sync属性。

然后的normal就是普通调用update了，它会调用queueWatcher，把当前watcher对象传入，这时会判断是否flush中，否时直接把watcher push到queue等待处理。否则拿到当前queue执行的index与watcher.id比较，然后排序插入。

接下来在nextTick执行flushSchedulerQueue。在这里就循环调用队列中所有watcher.run()

#### run
run主要的工作就是再次触发watcher.get函数调用getters，重复constructor默认的工作，如果watcher.get返回的值与原来不一样则替换新值，最后调用this.cb.call(this.vm, value, oldvalue)进行触发watch钩子

#### addDep
这个方法是提供给外部使用的，用于添加依赖，执行链比较深。

下面列举一下执行链：

各个地方需要绑定数据 -> defineReactive -> dep.depend -> watcher.addDep -> dep.addSub -> dep.subs.push(watcher)

最后是把defineReactive属性推到dep.subs队列中等待各个set与del或者Array更改时调用dep.notify。

#### evaluate
也是提供外部使用的，用于computed更新，这里也有一个很长的执行路径：

initState判断`vm.$options.computed`，并执行`initComputed(vm, vm.$options.computed)`。

到了initComputed函数，首先`for in`所有`computed与computed的key`，判断只要key如果不在vm中就执行`defineComputed(vm, key, userDef)`定义computed。

在defineComputed会经过ssr和是否需要cache判断，然后在`sharedPropertyDefinition.get`配置`createComputedGetter(key)`，最后把sharedPropertyDefinition给到`Object.defineProperty(vm,key,sharedPropertyDefinition)`进行配置。

createComputedGetter是一个高阶函数，返回computedGetter。

到了computedGetter这一步，其实就是返回到赋值给`sharedPropertyDefinition.get`那一步了。这一块逻辑主要是触发`watcher.evaluate`函数用的。

也就是说只要访问了computed[key]，就会触发`computed[key].get`，从而触发`computedGetter`函数，然后调用watcher.evaluate，最后调用`watcher.value = watcher.get()`更新。

![Watcher脑图](https://github.com/TinyScript/notes/blob/master/images/vue/Watcher.jpg)
