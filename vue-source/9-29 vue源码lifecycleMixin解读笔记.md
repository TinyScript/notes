#### lifecycle.js -- function lifecycleMixin
`lifecycleMixin`生命周期混入，主要是给Vue的原型注入了3个方法：
- Vue.prototype._update(`vnode: VNode`, `hydrating?: boolean`) -- 更新
- Vue.prototype.$forceUpdate() -- 强制更新
- Vue.prototype.$destory() -- 销毁

接下来细看各个方法都干了些什么。

#### lifecycle.js -- Vue.prototype._update

`_update`首先缓存原vm的`$el元素`与`_vnode虚拟元素`到`preEl`与`preVnode`。

在调用`_update`方法时，将当前组件传入到`setActiveInstance`，激活activeInstance的属性，并拿到一个给activeInstance变量`恢复到上一个实例`的handler。

将传入的vnode给到`vm._vnode`，然后操作缓存的`prevVnode`，如果没有缓存的虚拟节点，则进入初始化渲染，否则对缓存的虚拟节点进行更新。而`vm.__patch__`这里就不说了，篇幅太长专门留一个专题说。

完毕后调用刚刚的handler`restoreActiveInstance`，恢复到上一个被激活的实例。

如果存在上一个元素`prevEl`，则置空`prevEl.__vue__`，如果当前实例有$el，则将当前实例给到`vm.$el.__vue__ = vm`。

最后做HOC的判断（暂时没搞懂）。

总结：`_update`是为了针对传入的`vnode`进行初始化或者更新。
```javascript
export function lifecycleMixin (Vue: Class<Component>) {
  Vue.prototype._update = function (vnode: VNode, hydrating?: boolean) {
    const vm: Component = this
    const prevEl = vm.$el
    const prevVnode = vm._vnode
    const restoreActiveInstance = setActiveInstance(vm)
    vm._vnode = vnode
    // Vue.prototype.__patch__ is injected in entry points
    // based on the rendering backend used.
    if (!prevVnode) {
      // initial render
      vm.$el = vm.__patch__(vm.$el, vnode, hydrating, false /* removeOnly */)
    } else {
      // updates
      vm.$el = vm.__patch__(prevVnode, vnode)
    }
    restoreActiveInstance()
    // update __vue__ reference
    if (prevEl) {
      prevEl.__vue__ = null
    }
    if (vm.$el) {
      vm.$el.__vue__ = vm
    }
    // if parent is an HOC, update its $el as well
    if (vm.$vnode && vm.$parent && vm.$vnode === vm.$parent._vnode) {
      vm.$parent.$el = vm.$el
    }
    // updated hook is called by the scheduler to ensure that children are
    // updated in a parent's updated hook.
  }
  /**
    *
    * something code...
    *
    **/
```

#### lifecycle.js -- Vue.prototype.$forceUpdate

`$forceUpdate`顾明思议，强制更新。代码也很容易看懂，就是调用`this._watcher.update()`。

```javascript
Vue.prototype.$forceUpdate = function() {
    const vm:Component = this
    if(vm._watcher) {
        vm._watcher.update()
    }
}
```

### lifecycle.js -- Vue.prototype.$destory

`$destory`你懂的，就是销毁。

这个函数主要是做各种组件属性的拆卸，以及调用销毁时相关的钩子。

1. 调用销毁前的钩子函数`beforeDestroy`以及完成销毁后的钩子`destroyed`。
2. 从父节点删除本vm实例。
3. 拆卸vm._watcher。
4. 从`vm._data.__ob__`清理引用，也就是将vmCount归零。
5. `isDestroyed`标记这个是被destroyed销毁的节点。
6. `vm.__patch__(vm._vnode, null)`通过`__patch__`方法，不传第二个vnode参数，则会调用`invokeDestroyHook`销毁钩子，最后会判断传入的节点是否有子节点进行递归，进行整个节点树的销毁。
7. 最后调用销毁钩子`destroyed`，并取消所有的事件绑定
8. 清空`vm.$el.__vue__`，清空`vm.$vnode.parent`


```javascript
Vue.prototype.$destroy = function () {
  const vm: Component = this
  if (vm._isBeingDestroyed) {
    return
  }
  callHook(vm, 'beforeDestroy')
  vm._isBeingDestroyed = true
  // remove self from parent
  const parent = vm.$parent
  if (parent && !parent._isBeingDestroyed && !vm.$options.abstract) {
    remove(parent.$children, vm)
  }
  // teardown watchers
  if (vm._watcher) {
    vm._watcher.teardown()
  }
  let i = vm._watchers.length
  while (i--) {
    vm._watchers[i].teardown()
  }
  // remove reference from data ob
  // frozen object may not have observer.
  if (vm._data.__ob__) {
    vm._data.__ob__.vmCount--
  }
  // call the last hook...
  vm._isDestroyed = true
  // invoke destroy hooks on current rendered tree
  vm.__patch__(vm._vnode, null)
  // fire destroyed hook
  callHook(vm, 'destroyed')
  // turn off all instance listeners.
  vm.$off()
  // remove __vue__ reference
  if (vm.$el) {
    vm.$el.__vue__ = null
  }
  // release circular reference (#6759)
  if (vm.$vnode) {
    vm.$vnode.parent = null
  }
}
```