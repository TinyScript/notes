### 0x00 什么是Dep.target
这个属性从我一开始看源码就很疑惑，到底是干嘛的，有什么用，什么时候才会给它赋值。

这个问题直到2个月后才开始慢慢理解，目的是为了生成现阶段收集依赖用的全局观察者。当有需要收集依赖的时候，就会对`Dep.target`这个属性进行访问了。

那么，有哪些地方会设置Dep.target呢？又有哪些地方会收集依赖呢？我们先从内容量少的开始讲。

### 0x01 什么地方会收集依赖？

#### Dep类的提供的方法，dep.depend
在`Dep类`中的depend方法，如果存在可依赖目标，则会对dep实例进行收集。

```javascript
class Dep {
  depend () {
    if (Dep.target) {
      Dep.target.addDep(this)
    }
  }
}
```

#### computed的依赖

还有就是computed的get方法会进行收集依赖，有一个defineComputed方法定义双向绑定。实际的代码看起来有点乱，但是目的就是为了执行一句代码：
```javascript
sharedPropertyDefinition.get = createComputedGetter(key)
```

下面看看简化后defineComputed的代码：

```javascript
const sharedPropertyDefinition = {
  enumerable: true,
  configurable: true,
  get: noop,
  set: noop
}

function defineComputed (
  target: any,
  key: string,
  userDef: Object | Function
) {
  if (typeof userDef === 'function') {
    sharedPropertyDefinition.get = createComputedGetter(key)
  } else {
    sharedPropertyDefinition.get = userDef.get 
      ? userDef.cache !== false 
        ? createComputedGetter(key)
        : createGetterInvoker(userDef.get)
      : noop
  }
  Object.defineProperty(target, key, sharedPropertyDefinition)
}
```

上面只是介绍来源，实际上重点是下面的代码`createComputedGetter`，是一个高阶函数，主要就是收集key等待下一次get触发而调用。

如果有`Dep.target全局观察者`，并触发了`computed.get`，会把当前key对应的watcher里所有的dep都进行依赖。

```javascript
function createComputedGetter (key) {
  return function computedGetter () {
    const watcher = this._computedWatchers && this._computedWatchers[key]
    if (watcher) {
      if (Dep.target) {
        watcher.depend()
      }
      return watcher.value
    }
  }
}
```

#### 双向绑定的依赖

还有一个则是在`defineReactive`函数，在定义响应中的get会对依赖进行收集

```javascript
function defineReactive (
  obj: Object,
  key: string,
  val: any,
  customSetter?: ?Function,
  shallow?: boolean
) {
  const dep = new Dep()
  Object.defineProperty(obj, key, {
    enumerable: true,
    configurable: true,
    get: function reactiveGetter() {
      if (Dep.target) {
        dep.depend()
      }
    }
  })
}
```

### 0x02 总结：
1. 在调用dep.depend时会校验Dep.target
2. 在defineComputed的get会校验Dep.target
3. 在defineReactive的get会校验Dep.target

因为会校验Dep.target，说明这地方就有可能需要进行依赖了。

于是全局搜索了一下depend方法，发现确实是如此，可以总结出只要访问了就会对它们进行依赖收集。

