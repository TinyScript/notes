### 0x00 什么地方会收集依赖
依赖收集是只有在发生`对象属性`访问的时候，才会触发，并且Dep.target是有东西的情况下，访问器get函数就会触发对应的depend。

### 0x01 什么情况下会访问对象属性？
1.比如在组件`挂载`时，会触发`updateComponent函数`，里面会有一个`this._update(this._render(), hydrating)`。

在执行this._render时，就会对vue文件里的template进行渲染，渲染的过程中必然会访问`对象属性`，从而触发`get访问器`

2.还有就是watcher是deep状态时，只要执行了`watcher.get`都会对它的值进行traverse一遍，也就是所有对象属性都会进行访问。

### 0x02 Dep.target什么时候会有东西？
1. 组件mount时
2. watcher.get时
3. 执行callHook函数时（这时pushTarget的是undefined）

### 0x03 收集依赖的时机
收集依赖的时机分两种，**属性**与**计算属性**。

##### 1. 属性
属性基本是在挂载阶段会进行收集，挂载阶段执行this._render合成页面模板时，会对模板所需要的属性进行访问。

此时因为挂载阶段进行了`pushTarget`，所以`Dep.target`就是当前vue组件的watcher，只要对属性访问，都会触发每个属性的`dep.depend()`与`childOb.dep.depend()`，如果还有数组，则会对数组对象进行收集`dependArray(value)`。

##### 2. 计算属性
计算属性同样在模板被访问时，会触发访问器的computedGetter。

如果先前computed被变更过（执行watcher.update），那么computed的watcher.dirty会被标记为脏。

在computedGetter会拿到当前vue实例watchers列表所对应key的watcher。

如果此时watcher.dirty被标记为脏，则执行watcher.evaluate，最后会执行到`watcher.getter`这个函数，就是我们在外面注册的computed，此时会将函数内部所使用过的属性进行收集。

最后如果是在挂载阶段，computed的watcher会把所有收集到的依赖，上报给vue组件的watcher。`watcher.depend`。

![watcher依赖收集](https://github.com/TinyScript/notes/blob/master/images/vue/watcher%E7%9A%84%E4%BE%9D%E8%B5%96%E6%94%B6%E9%9B%86.jpg)
