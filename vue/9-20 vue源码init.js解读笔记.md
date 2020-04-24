今天开始了Vue源码的解读，从入口`src/core/instance/index.js`开始。

Vue构造函数并没有其他东西，作者主要的工作还是把Vue注入到几个mixin函数中。
1. initMixin(Vue) - `init.js` 初始化
2. stateMixin(Vue) - `state.js` 状态
3. eventsMixin(Vue) - `events.js` 事件
4. lifecycleMixin(Vue) - `lifecycle.js` 生命周期
5. renderMixin(Vue) - `render.js` 渲染

#### init.js -- function initMixin

从initMixin开始看，以下是关键代码。初始化时vm就是当前组件，每次都会新建一个uid，如果是销毁了再重新创建，uid也是不一样的，除非使用keep-alive进行缓存，然后就是是各种的初始化，再将vm实例注入。
```javascript
let uid = 0

Vue.prototype._init = function() {
    const vm = this;
    vm._uid = uid++
    /** 
     * something code...
     */
    if(process.env.NODE_ENV !== 'production') {
        initProxy(vm)
    } else {
        vm._renderProxy = vm
    }
    vm._self = vm
    initLifecycle(vm)
    initEvents(vm)
    initRender(vm)
    callHook(vm, 'beforeCreate')
    initInjections(vm) // resolve injections before data/props
    initState(vm)
    initProvide(vm) // resolve provide after data/props
    callHook(vm, 'created')
}
```

#### proxy.js -- function initProxy

如果不是在production环境下，renderProxy用的是Proxy对象，如果没有Proxy则直接用vm实例，production环境用的也是vm实例。

```javascript 
function initProxy (vm) {
    if (hasProxy) {
      // determine which proxy handler to use
      const options = vm.$options
      const handlers = options.render && options.render._withStripped
        ? getHandler
        : hasHandler
      vm._renderProxy = new Proxy(vm, handlers)
    } else {
      vm._renderProxy = vm
    }
}
```

继续往下看初始化`initLifecycle`。

#### lifecycle.js -- function initLifecycle

看名字这里应该就是初始化生命周期用的。这里的`$options.parent`被抽出到`$parent`属性了，然后初始化了$root属性，还有`$children[]`与`$refs{}`，以及监听本实例的`_watcher`实例，判断keep-alive是否被激活的`_incative`与`_directInactive`，`_isMounted`是否被挂载，`_isDestoryed`是否销毁和`_isBeingDestoryed`正在销毁
```javascript
function initLifecycle(vm) {
    const options = vm.$options
    let parent = options.parent
    
    if (parent && !options.abstract) {
        while (parent.$options.abstract && parent.$parent) {
          parent = parent.$parent
        }
        parent.$children.push(vm)
    }
    
    vm.$parent = parent
    vm.$root = parent ? parent.$root : vm
    
    vm.$children = []
    vm.$refs = {}

    vm._watcher = null
    vm._inactive = null
    vm._directInactive = false
    vm._isMounted = false
    vm._isDestroyed = false
    vm._isBeingDestroyed = false
}
```

#### events.js -- function initEvents

`initEvents`初始化了`_events`，`_hasHookEvent`，还有当前实例存在父级监听，则更新监听。而`updateComponentListeners`则调用了vdom的更新监听函数`updateListeners`，vdom的更新监听，主要是对新老监听和是否一次性执行调用做处理。
```javascript
export function initEvents (vm: Component) {
  vm._events = Object.create(null)
  vm._hasHookEvent = false
  // init parent attached events
  const listeners = vm.$options._parentListeners
  if (listeners) {
    updateComponentListeners(vm, listeners)
  }
}

export function updateComponentListeners (
  vm: Component,
  listeners: Object,
  oldListeners: ?Object
) {
  target = vm
  updateListeners(listeners, oldListeners || {}, add, remove, createOnceHandler, vm)
  target = undefined
}
```

#### render.js -- function initRender

`initRender`初始化了以下属性：
1. `_vnode`、`_staticTrees`。
2. `parentVnode`赋值给`$vnode`，所以`$vnode.context`又是`$parent`。
3. `$slots`属性通过`core/instance/render-helps/resolve-slots.js`的`resolveSlots`函数进行初始化，会用到`$options._renderChildren`和`options._parentVnode.context`两个属性。
4. `$scopedSlots`则是冻结对象
5. 拿到`$vnode.data`和`$options._parentListeners`再对`vm.$attrs`和`vm.linsteners`进行响应初始化`defineReactive`。

```javascript
export function initRender (vm: Component) {
  vm._vnode = null 
  vm._staticTrees = null 
  const options = vm.$options
  const parentVnode = vm.$vnode = options._parentVnode 
  const renderContext = parentVnode && parentVnode.context
  vm.$slots = resolveSlots(options._renderChildren, renderContext)
  vm.$scopedSlots = emptyObject
  vm._c = (a, b, c, d) => createElement(vm, a, b, c, d, false)
  vm.$createElement = (a, b, c, d) => createElement(vm, a, b, c, d, true)

  const parentData = parentVnode && parentVnode.data

  if (process.env.NODE_ENV !== 'production') {
    defineReactive(vm, '$attrs', parentData && parentData.attrs || emptyObject, () => {
      !isUpdatingChildComponent && warn(`$attrs is readonly.`, vm)
    }, true)
    defineReactive(vm, '$listeners', options._parentListeners || emptyObject, () => {
      !isUpdatingChildComponent && warn(`$listeners is readonly.`, vm)
    }, true)
  } else {
    defineReactive(vm, '$attrs', parentData && parentData.attrs || emptyObject, null, true)
    defineReactive(vm, '$listeners', options._parentListeners || emptyObject, null, true)
  }
}
```

#### observer.js -- function defineReactive

`defineReactive`函数从名字上看是定义响应的意思，这里一开始就出现了看不懂的`Dep`类，以及`dep.depend`方法，不过暂时不影响本函数的主要作用，可以在后续的`Dep.js`深入。首先通过`Object.getOwnPropertyDescriptor`拿到`obj[key]`的descriptor描述对象，如果存在对象并且可编辑，则拿到对象的`get和set函数`共后续使用。

如果没有`get函数`或者有`set函数`，并且只传了obj与key的情况下，通过obj[key]拿到val备用。如果不是浅定义的话，将val收集到`observe`监听，这个函数比价复杂，在后续`Observer.js`再深入研究。

前面做的铺垫，就是为了我们常说的`Object.definedProperty`函数了，这里的get通过之前拿的`getter函数`拿到value并返回，其中还调用了`dep.depend`，数组则用了数组专用的`dependArray`函数。而`setter函数`，最终是为了修改`obj[key]`并做`dep.notify`通知作用，期间做了一些相同的值阻止刷新，如果是新的值还会重新被`observe`观察。

这个函数主要就是给obj下的属性定义新的依赖值，形成双向绑定。

#### 代码太长我就不贴了。

#### lifecycle.js -- function callHook

`callHook`顾名思义调用钩子，在初始化阶段调用了两次钩子：`callHook(vm, 'beforeCreated')`和`callHook(vm, 'created')`。

callHook函数开始和结束都会各触发一次`pushTarget`与`popTarget`函数，用于配合Dep类实际上干啥暂时还不知道。

拿到hook名后会从vm.$options[hook]拿到生命周期的钩子，通过`invokeWithErrorHandling`函数调用钩子，如果存在钩子事件，则emit发射hooks事件。

```javascript
export function callHooks(vm:Component, hook: string) {
    pushTarget()
    const handlers = vm.$options[hook]
    const info = `${hook} hook`
    if(handlers) {
        for(var i = 0, len = handlers.length; i < len; i++) {
            invokeWithErrorHandling(handlers[i], vm, null, vm, info)
        }
    }
    if (vm._hasHookEvent) {
        vm.$emit('hook:' + hook)
    }
    popTarget()
}
```

#### inject.js -- function initInjections

`initInjections`初始化注入，会拿到`$options.inject`与`vm`调用resolveInject函数，从provide取到值给result，如果取不到值就没啥事了，如果成功取到值，初始调用`toggleObserving`关闭observe观察，然后对解析inject拿到的所有key进行`defineReactive`监听，然后再打开observe。

```javascript
export function initInjections (vm: Component) {
  const result = resolveInject(vm.$options.inject, vm)
  if (result) {
    toggleObserving(false)
    Object.keys(result).forEach(key => {
      /* istanbul ignore else */
      if (process.env.NODE_ENV !== 'production') {
        defineReactive(vm, key, result[key], () => {
          warn(
            `Avoid mutating an injected value directly since the changes will be ` +
            `overwritten whenever the provided component re-renders. ` +
            `injection being mutated: "${key}"`,
            vm
          )
        })
      } else {
        defineReactive(vm, key, result[key])
      }
    })
    toggleObserving(true)
  }
}
```

#### inject.js -- function resolveInject

拿到`$options.inject`与`vm`，然后从$options.inject拿到所有的key，再对所有的inject进行处理。

如果有vm的话，从inject拿到form作为provideKey，在判断vm是否有`source._provided`，`provideKey`是不是provided的内部属性，命中了的话则给`result[key]`记录，如果找完了当前的vm没有找到，就往vm.$parent去找，直到根节点为止。

如果没有vm，则去`inject[key].default`找provide然后给到`result[key]`，如果没有，在开发环境报警告

```javascript
export function resolveInject (inject: any, vm: Component): ?Object {
  if (inject) {
    // inject is :any because flow is not smart enough to figure out cached
    const result = Object.create(null)
    const keys = hasSymbol
      ? Reflect.ownKeys(inject)
      : Object.keys(inject)

    for (let i = 0; i < keys.length; i++) {
      const key = keys[i]
      // #6574 in case the inject object is observed...
      if (key === '__ob__') continue
      const provideKey = inject[key].from
      let source = vm
      while (source) {
        if (source._provided && hasOwn(source._provided, provideKey)) {
          result[key] = source._provided[provideKey]
          break
        }
        source = source.$parent
      }
      if (!source) {
        if ('default' in inject[key]) {
          const provideDefault = inject[key].default
          result[key] = typeof provideDefault === 'function'
            ? provideDefault.call(vm)
            : provideDefault
        } else if (process.env.NODE_ENV !== 'production') {
          warn(`Injection "${key}" not found`, vm)
        }
      }
    }
    return result
  }
}
```


#### function initState
`initState`初始化vue的几个关键属性
- 初始化`_watchers`队列
- 有`$options.props`初始化`initProps`
- 有`$options.methods`初始化`initMethods`
- 有`$options.data`，初始化`initData`
- 有`$options.computed`，初始化`initComputed`
- 有`$options.watch`，初始化`initWatch`

```javascript
export function initState (vm: Component) {
  vm._watchers = []
  const opts = vm.$options
  if (opts.props) initProps(vm, opts.props)
  if (opts.methods) initMethods(vm, opts.methods)
  if (opts.data) {
    initData(vm)
  } else {
    observe(vm._data = {}, true /* asRootData */)
  }
  if (opts.computed) initComputed(vm, opts.computed)
  if (opts.watch && opts.watch !== nativeWatch) {
    initWatch(vm, opts.watch)
  }
}
```

#### function initProvide
`initProvide`初始化提供者，从`$options.provide`给到`vm._provided`

```javascript
export function initProvide (vm: Component) {
  const provide = vm.$options.provide
  if (provide) {
    vm._provided = typeof provide === 'function'
      ? provide.call(vm)
      : provide
  }
}
```

#### 最后到initMixin挂载部分

非`production`阶段，如果需要`config.performance`性能监控并且要`mark`标记，则做响应处理。

最后如果配置中写有`$options.el`，则挂载到页面中el对应的元素标签中。