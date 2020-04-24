### 0x00 pushTarget
在Dep.js会对外提供此方法，用于控制当前的全局观察者Dep.target，也就是watcher对象。

```javascript
function pushTarget (target: ?Watcher) {
  targetStack.push(target)
  Dep.target = target
}
```

### 0x01 popTarget
这个方法则反之。

### 0x02 正文
目前发现Dep.target会在几个地方出现赋值：

#### 1. callHook
没错，每个生命周期都会触发pushTarget，所以一趟下来你有几个组件就会往里面怼几次pushTarget。如果是树形层级很深，那到叶子节点的组件，就也会有很深的targetStack。

当然Dep.target依旧只有一个，就当前激活的依赖收集对象。这里只是顺手写一下，~~因为根本不会pushTarget内容~~。

```javascript
export function callHook (vm: Component, hook: string) {
  pushTarget()
  const handlers = vm.$options[hook]
  const info = `${hook} hook`
  if (handlers) {
    for (let i = 0, j = handlers.length; i < j; i++) {
      invokeWithErrorHandling(handlers[i], vm, null, vm, info)
    }
  }
  if (vm._hasHookEvent) {
    vm.$emit('hook:' + hook)
  }
  popTarget()
}
```

#### 2. 组件的watcher.get
当beforeCreate、created、beforeMount钩子都完成，就差开始挂载了。

于是就要开始创建当前组件的观察者了，因为这个watcher会开始处理vdom，以及组件本身的数据。

所以当前组件需要一个专门的观察者对他们进行数据的依赖收集，方便之后管理。

一般new Vue之后在完成挂载popTarget之前，首层组件生成的watcher会一直存在targetStack里。再根据组件层级的深度targetStack会继续延长。

#### 3. computed的watcher.get
组件的watcher都开始上了，那肯定会执行watcher的this.getter，组件的this.getter则是updateComponent，直到this._render执行后，computed的属性被访问了，于是就触发了computed的getter。

主线来了，在触发computed的getter过程中，就触发了`pushTarget`，改变了当前正在激活的`Dep.target`依赖目标。

也就是说每次触发computed的getter都会激活当前的`Dep.target = watcher`。

#### 4. Vue.prototype.$watch的watcher.get
在用$watch绑定时也会激活当前的watcher对象，给targetStack路径再加一层watcher对象。

#### 5. 在赋值时
赋值的时候，会触发属性的reactiveSetter中的`dep.notify`方法。notify会去触发`watcher.update`，然后就回到了watcher那一张的执行路径

```javascript
dep.notify()
watcher.update()
queueWatcher(this) // 如有新加入的会排序然后再flush
nextTick(flushSchedulerQueue)
// ======直到下一帧======
watcher.run()
watcher.get()
pushTarget(this)
this.getter.call(vm, vm)
get: proxyGetter
get: reactiveGetter
```

在赋值时触发的notify也会唤醒观察者到Dep.target去。

需要注意的是，并不是watcher本身会触发，而是触发watcher.get函数才会执行pushTarget函数。watcher.run也会触发watcher.get