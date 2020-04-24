#### events.js -- function eventsMixin

`eventsMixin`事件混入函数，主要就是绑定了以下四个事件：
- $on
- $once
- $off
- $emit

```javascript
export function eventsMixin (Vue: Class<Component>) {
  const hookRE = /^hook:/
  Vue.prototype.$on = on;
  Vue.prototype.$once = once;
  Vue.prototype.$off = off;
  Vue.prototype.$emit = emit;
}
```

`$on`方法，绑定事件的方法，如果首个参数是数组，则对vm进行循环$on绑定。如果不是数组，则存到`vm._events`中，如果events是字符串并且含有`hooks:`字符。则标记为钩子事件`_hasHookEvent`。

```javascript
Vue.prototype.$on = function (event: string | Array<string>, fn: Function): Component {
  const vm: Component = this
  if (Array.isArray(event)) {
    for (let i = 0, l = event.length; i < l; i++) {
      vm.$on(event[i], fn)
    }
  } else {
    (vm._events[event] || (vm._events[event] = [])).push(fn)
    if (hookRE.test(event)) {
      vm._hasHookEvent = true
    }
  }
  return vm
}
```

`$once`比较简单
```javascript
Vue.prototype.$once = function (event: string, fn: Function): Component {
  const vm: Component = this
  function on () {
    vm.$off(event, on)
    fn.apply(vm, arguments)
  }
  on.fn = fn
  vm.$on(event, on)
  return vm
}
```

`$off`与`$on`相反，不同的是如果只调用$off的话，会解除所有的事件。代码就不贴了。

`$emit`则是事件派发，类似jquery的trigger。根据事件名对之前绑定过的events进行调用。

event必须是字符串，这个字符串会作为一个key在vm._events对象取到callbacks，如果存下cbs，则进行invokeWithErrorHandling调用。

```javascript
Vue.prototype.$emit = function (event: string): Component {
  const vm: Component = this
  let cbs = vm._events[event]
  if (cbs) {
    cbs = cbs.length > 1 ? toArray(cbs) : cbs
    const args = toArray(arguments, 1)
    const info = `event handler for "${event}"`
    for (let i = 0, l = cbs.length; i < l; i++) {
      invokeWithErrorHandling(cbs[i], vm, args, vm, info)
    }
  }
  return vm
}
```

#### util/error.js -- function invokeWithErrorHandling

`invokeWithErrorHandling`主要就是为了统一捕获函数执行时的错误。
- 捕获函数执行时如果`handler`传`非函数`的错误
- 如果函数执行期间报错
- 并对promise进行了处理，并cache promise的报错，如果是promise的报错则提供`res._handled=true`的标识

```javascript
export function invokeWithErrorHandling (
  handler: Function,
  context: any,
  args: null | any[],
  vm: any,
  info: string
) {
  let res
  try {
    res = args ? handler.apply(context, args) : handler.call(context)
    if (res && !res._isVue && isPromise(res) && !res._handled) {
      res.catch(e => handleError(e, vm, info + ` (Promise/async)`))
      // issue #9511
      // avoid catch triggering multiple times when nested calls
      res._handled = true
    }
  } catch (e) {
    handleError(e, vm, info)
  }
  return res
}
```

总结，eventMixin只是混入了4个事件，对$emit的调用做了通用错误处理，扔进`invokeWithErrorHandling`方法进行调用了。