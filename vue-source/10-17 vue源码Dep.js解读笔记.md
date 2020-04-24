### 0x00 class -- Dep

今天来看看`Dep`依赖收集类。

`Dep`类定义了一个静态变量`target`，还有两个成员变量`id`和`subs`。以下是变量的规范：

- target用来引用Watcher类的实例。
- id用来表示当前Dep实例的唯一标识（不严谨）。
- subs则是订阅者列表。

### 0x01 三个变量：id、subs、target

在新建的时候会初始化`subs`与自增`id`，而`target`在文件定义了`class Dep`后就会默认设置为null。

```javascript
import type Watcher from './watcher'
import { remove } from '../util/index'
import config from '../config'

let uid = 0

export default class Dep {
  // something code...
}

Dep.target = null
```

### 0x02 constructor

在new Dep时会初始化`id`与`subs`两个成员变量。

```javascript
constructor() {
    // uid在最开头就定义了，Dep做实例唯一标示用
    this.id = uid++
    this.subs = []
}
```

### 0x03 Dep就是订阅发布模式的Depend + Publish

然后就是订阅发布模式的一些方法了，不同的是Public类合并到了`Depend类`。

而传入的`Watcher类`实例，对应的就是`Subscribe类`的实例了，里边肯定会带一个`update`方法，用于给发布者循环调用进行通知。

然后还有一个depend，如果`Dep.target`处于有`Watcher`的状态下，则给`Watcher`实例添加一个`Dep依赖实例(this)`。
```javascript
addSub(sub: Watcher) {
    this.subs.push(sub)
}
removeSub(sub: Watcher) {
    remove(this.sub, sub)
}
notifiy() {
    const subs = this.subs.slice()
    for(let i = 0, l = subs.length; i < l; i++) {
        subs[i].update()
    }
}
depend() {
    if(Dep.target) {
        Dep.target.addDep(this)
    }
}
```

### 0x04 pushTarget与popTarget

这两个方法用来明确哪个Watcher正在处理中的状态。

底部提供出了两个方法`pushTarget`与`popTarget`，用来存储当前`Dep类`正在处理的是哪个Watcher，以及处理完后下一个Watcher是谁。

```javascript
let uid = 0
export default class Dep { }
Dep.target = null
const targetStack = []

export function pushTarget(target: Watcher) {
    targetStack.push(targetStack)
    Dep.target = target
}

export function popTarget() {
    targetStack.pop()
    Dep.target = targetStack[targetStack - 1]
}
```

### 0x05 总结：

`Dep.js`文件主要有三个概念：
1. 订阅发布模式中的`收集订阅者`和`广播发布`两部分。
2. `Dep.target`确定当前正在处理的Watcher是谁，并且用`targetStack`进行排队处理。
3. 如果当前有确定正在处理的Watcher实例，可以通过`depend`方法给Watcher实例添加依赖。