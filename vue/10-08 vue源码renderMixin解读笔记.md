#### 0x00 render.js -- function renderMixin

目前是到了最后一个mixin了，这个混入主要做了3点：
- installRenderHelpers(Vue.prototype)
- Vue.prototype.$nextTick
- Vue.prototype._render

`installRenderHelpers`主要是给Vue.prototype添加了许多渲染方法。

`$nextTick`则是对外提供了内部封装好的nextTick的方法。

`_render`最终会返回一个vnode，由vm.$options.render渲染生成后的vnode。

```javascript
export function renderMixin(Vue: Class<Component>) {
    installRenderHelpers(Vue.prototype)
    Vue.prototype.$nextTick = function (fn: Function) {
        return nextTick(fn, this)
    }
    Vue.prototype._render = function (fn: Function) {
        // something code...
    }
}
```

#### 0x01 instance/render-helpers/index.js -- function installRenderHelpers

`installRenderHelpers`，会给Vue.prototype绑定许多渲染方法。这里贴个代码，给缩写的变量名做个字典就行了。

```javascript
export function installRenderHelpers (target: any) {
  target._o = markOnce
  target._n = toNumber
  target._s = toString
  target._l = renderList
  target._t = renderSlot
  target._q = looseEqual
  target._i = looseIndexOf
  target._m = renderStatic
  target._f = resolveFilter
  target._k = checkKeyCodes
  target._b = bindObjectProps
  target._v = createTextVNode
  target._e = createEmptyVNode
  target._u = resolveScopedSlots
  target._g = bindObjectListeners
  target._d = bindDynamicKeys
  target._p = prependModifier
}
```

#### 0x02 core/util/index.js -- Vue.prototype.$nextTick

`$nextTick`里包裹这一个`nextTick`函数，主要讲的就是nextTick函数，这里有点像是Promise的处理方式，将nextTick给到的`function`和`context`用函数包装起来，给到callbacks数组待用。

包装函数内部的逻辑则是判断有无`cb`函数，有则call，没有则新建一个promise，将`ctx`给到promise的resolve。提供给`$nextTick().then(ctx => { console.log(ctx) })`使用。

render.js给Vue注入的$nextTick，包裹着nextTick函数。
```javascript
Vue.prototype.$nextTick = function (fn: Function) {
  return nextTick(fn, this)
}
```
utils.js中的nextTick函数。
```javascript
export function nextTick (cb?: Function, ctx?: Object) {
  let _resolve
  callbacks.push(() => {
    if (cb) {
      try {
        cb.call(ctx)
      } catch (e) {
        handleError(e, ctx, 'nextTick')
      }
    } else if (_resolve) {
      _resolve(ctx)
    }
  })
  if (!pending) {
    pending = true
    timerFunc()
  }
  // $flow-disable-line
  if (!cb && typeof Promise !== 'undefined') {
    return new Promise(resolve => {
      _resolve = resolve
    })
  }
}

```

#### 0x03 render.js -- Vue.prototype._render

`_render`主要负责渲染节点，如果vm.$options存在_parentVnode，则会给vm.$scopedSlots初始化一个基本的slots格式。

然后将_parentVnode给到$vnode中，`vm.$vnode = _parentVnode`。

然后新建一个`vnode`变量开始渲染自己，首先会激活`currentRenderingInstance`变量，顾名思义当前渲染中的实例，用于告诉外部谁正在渲染中。

然后会用`vm.$options.render`的方法，渲染后将值赋给刚刚创建的`vnode`变量。如果render不是一个函数，则将vm._vnode赋值给到`vnode`变量。最后解除当前渲染中实例的锁currentRenderingInstance。

如果`vnode`是数组并且长度唯一时，直接取第一个。

最后对`vnode`变量进行处理，如果不是`VNode`类派生，则给`vnode`变量创建一个空的VNode实例。

最后将_parentVnode给到vnode.parent，`vnode.parent = _parentVnode`，并返回处理后的这个`vnode`变量。

```javascript
Vue.prototype._render = function (): VNode {
  const vm: Component = this
  const { render, _parentVnode } = vm.$options

  if (_parentVnode) {
    vm.$scopedSlots = normalizeScopedSlots(
      _parentVnode.data.scopedSlots,
      vm.$slots,
      vm.$scopedSlots
    )
  }

  vm.$vnode = _parentVnode
  // render self
  let vnode
  try {
    currentRenderingInstance = vm
    vnode = render.call(vm._renderProxy, vm.$createElement)
  } catch (e) {
    handleError(e, vm, `render`)
    /* istanbul ignore else */
    if (process.env.NODE_ENV !== 'production' && vm.$options.renderError) {
      try {
        vnode = vm.$options.renderError.call(vm._renderProxy, vm.$createElement, e)
      } catch (e) {
        handleError(e, vm, `renderError`)
        vnode = vm._vnode
      }
    } else {
      vnode = vm._vnode
    }
  } finally {
    currentRenderingInstance = null
  }
  // if the returned array contains only a single node, allow it
  if (Array.isArray(vnode) && vnode.length === 1) {
    vnode = vnode[0]
  }
  // return empty vnode in case the render function errored out
  if (!(vnode instanceof VNode)) {
    if (process.env.NODE_ENV !== 'production' && Array.isArray(vnode)) {
      warn(
        'Multiple root nodes returned from render function. Render function ' +
        'should return a single root node.',
        vm
      )
    }
    vnode = createEmptyVNode()
  }
  // set parent
  vnode.parent = _parentVnode
  return vnode
}
```

### 0x04 next-tick.js
`$nextTick`来自这个文件导出的`function nextTick`。

这个文件核心有如下几个：
1. 调用nextTick会给callbacks列表插入一个执行回调的函数，并修改MutationObserver监听的文本节点，使其触发更新。
2. MutationObserver会监听一个文本节点，节点只要更新就会执行传入的回调函数。
3. 回调中会执行callbacks列表中所有函数，并清空callbacks列表。

```javascript

export let isUsingMicroTask = false

const callbacks = []
let pending = false

function flushCallbacks () {
  pending = false
  const copies = callbacks.slice(0)
  callbacks.length = 0
  for (let i = 0; i < copies.length; i++) {
    copies[i]()
  }
}

let timerFunc

if (!isIE && typeof MutationObserver !== 'undefined' && (
  isNative(MutationObserver) ||
  // PhantomJS and iOS 7.x
  MutationObserver.toString() === '[object MutationObserverConstructor]'
)) {
  // Use MutationObserver where native Promise is not available,
  // e.g. PhantomJS, iOS7, Android 4.4
  // (#6466 MutationObserver is unreliable in IE11)
  let counter = 1
  const observer = new MutationObserver(flushCallbacks)
  const textNode = document.createTextNode(String(counter))
  observer.observe(textNode, {
    characterData: true
  })
  timerFunc = () => {
    counter = (counter + 1) % 2
    textNode.data = String(counter)
  }
  isUsingMicroTask = true
}

export function nextTick (cb?: Function, ctx?: Object) {
  callbacks.push(() => {
    cb.call(ctx)
  })
  if (!pending) {
    pending = true
    timerFunc()
  }
}
```

### 0x05 MutationObserver

这是一个DOM树变更观察者，专门用来监听指定元素下的树结构是否有改变。

用法：

- 创建MutationObserver实例，并传入会被触发的回调函数
- 实例调用observe方法，监听指定的`节点树`，并传入监听配置
- [监听配置表可以戳这里 - MDN-Observe监听配置](https://developer.mozilla.org/zh-CN/docs/Web/API/MutationObserverInit)

```javascript
var pending = false
var counter = 1

var targetNode = document.createTextNode(String(counter));

// 监听配置
var config = {
    attributes: true,
    childList: true,
    subtree: true,
    characterData: true
};
var callbacks = []

var mutationCallback = (mutationsList) => {
  pending = false
  console.log('mutationObserver')
  var copies = callbacks.slice(0)
  callbacks.length = 0
  for(var i = 0, l = copies.length; i < l; i++) {
    copies[i]()
  }
};

var observer = new MutationObserver(mutationCallback);

observer.observe(targetNode, config);

function nextTick() {
  console.log('a函数')
  callbacks.push(() => { console.log(123123) })
  if(!pending) {
    pending = true
    counter = (counter + 1) % 2
    targetNode.data = String(counter)
  }
}

```

这里只做了三件事，总结：
1. 安装render时需要的工具函数。
2. $nextTick下一次渲染函数处理。
3. _render，根据`vm.$options.render`渲染vnode，说实话看不太懂。可能是因为这里只是渲染混入的原因，没真正的使用到。