# 组件的watcher、$watch的watcher、computed的watcher源码逻辑记录：

### 组件：
vue在每次给data定义的属性赋值时，都会触发Object.defineProperty的set拦截器，修改被赋值的value，然后调用当前属性的对应的dep.nofity函数，notify会让所有观察这个属性的watcher对象进行watcher.update。

watcher.update执行后，会调用queueWatcher，将观察者塞入一个数组然后交给异步队列，等到主线程所有赋值都执行完毕以后再处理watcher的更新。

### watch：

到了异步队列，会遍历处理watcher数组，$watch的watcher和vue组件watcher会在run()的时候调用callback时更新。

### computed：
而computed的watcher则会在vue组件的watcher执行callback后，执行updateComponent时，所有模板上的属性都会被get拦截器拦截，包括computed在模板上的属性也会触发自己单独的get拦截器进行更新。

所以computed是一次被动的更新，需要等待组件watcher执行callback更新后，computed的watcher才会更新。