#### 0x00 property -- vm.\_\_patch\_\_

简要笔记：
1. `vm.__patch__`来自`Vue.prototype.__patch__`的实例化
2. `Vue.prototype.__patch__ = patch`来自`platform/web/runtime/index.js`文件
3. `export const patch = createPatchFunction({nodeOps, module})`来自`platform/web/runtime/patch.js`文件
4. `createPatchFunction`来自`core/vdom/patch.js`文件

```javascript
export function createPatchFunction(backend) {
    // something code ...
    return function patch(oldVnode, vnode, hydrating, removeOnly) {
        // something code ...
        return vm.elm
    }
}
```
5. 所以vm.patch就是经过一系列处理得到新的一个vm.elm元素，就是为更新`vm.$el`服务的
6. 但也不止于此，因为domdiff算法也在这个函数里头，所以必须开一个专题来做笔记，在此先mark一下。
7. 下面进入主题

---

### 0x01 vm.__patch__是什么，createPatchFunction又是什么

`vm.__patch__`函数来源于`createPatchFunction`，而`cretePatchFunction`是一个高阶函数（废话），但是这是一个近800行代码的高阶函数，里面有近700行的前置代码，最后才返回的`patch函数`提供给外部使用。

听起来就很夸张，一开始我也是很抗拒看这段代码，后来认真一看，这些前置函数都是一些方法，为返回出去的`patch函数`服务的，所以我们可以直接看patch函数都干了些啥。

首先看`patch函数`，传入了四个参数：
1. `oldVnode` 老虚拟节点
2. `vnode` 新虚拟节点
3. `hydrating` 不知道
4. `removeOnly` 也不知道

最后返回了一个`vnode.elm`虚拟节点的元素，可以先理解成是给新节点打补丁？
```javascript
function patch(oldVnode, vnode, hydrating, removeOnly) {
    // something code ...
    return vnode.elm  
}
```

### 0x02不传vnode时patch会干什么

往下看第一段代码，针对`新虚拟节点`做了一次判断，如果为空，就阻止往下跑了，然后判断`老虚拟节点`的参数位是否有传入，有传入的话调用`invokeDestoryHook(oldVnode)`，对老虚拟节点进行销毁处理。

```javascript
function patch(oldVnode, vnode, hydrating, removeOnly) {
    if (isUndef(vnode)) {
      if (isDef(oldVnode)) invokeDestroyHook(oldVnode)
      return
    }
    // something code...
    return vnode.elm  
}
```

### 0x03 不传oldVnode时patch在干什么

继续往下看，初始化了两个变量，`isInitailPatch`和`insertedVnodeQueue`，分别为`是否处于初始化patch状态`和`需要被插入的队列`。

然后进入判断，如果`oldVnode`没有传入的情况下，就让`isInitailPatch`标记为初始化状态，然后调用`createElm(vnode, insertedVnodeQueue)`函数，对vnode进行创建。

else部分的内容下边再讲。

最后就调用插入钩子，`invokeInsertHook(vnode, isertedVnodeQueue, isInitailPatch)`对创建好的vnode进行处理，然后返回处理（创建）好的vnode。

总结：在`oldVnode参数位`没有传内容时，这部分就是给虚拟节点创建元素用的，然后将`createElm函数`给`vnode`新创建的元素返回出去。

```javascript
function patch(oldVnode, vnode, hydrating, removeOnly) {
    if (isUndef(vnode)) {
      if (isDef(oldVnode)) invokeDestroyHook(oldVnode)
      return
    }
    
    let isInitialPatch = false
    const insertedVnodeQueue = []
    
    if (isUndef(oldVnode)) {
      // empty mount (likely as component), create new root element
      isInitialPatch = true
      createElm(vnode, insertedVnodeQueue)
    } else {
      // something code...
    }
    invokeInsertHook(vnode, insertedVnodeQueue, isInitailPatch)
    
    return vnode.elm   
}
```

### 0x04 三个生僻函数

刚刚出现了三个生僻函数`invokeDestroyedHook`、`createElm`、`invokeInsertHook`，正是在`createPatchFunction`高阶函数下提供给`patch`使用的工具函数。看名字能猜个大概：

1. invokeDestroyedHook 调用销毁钩子
2. invokeInsertedHook 调用插入钩子
3. createElm 给vnode创建元素`vnode.elm`

#### 0x05 function invokeDestoryedHook

`invokeDestoryedHook`这个熟，之前有讲过。这个函数是为了销毁vnode用的，如果vnode有`data`和`children`属性，就会对vnode做文章。

如果有`vnode.data`，则会找到`let i = vnode.data.hook.destory`的销毁钩子函数进行调用`i(vnode)`，然后对`cbs.destory[index](vnode)`所有销毁函数进行调用，继续销毁vnode。

接下来继续往下看，如果有`vnode.children`则对所有的子虚拟节点进行递归销毁`invokedDestoryHook(vnode.children[j])`

总结：这里就是为了销毁vnode可能存在内存泄露而做的处理。

这里留下了两个问题：
1. `vnode.data.hook.destory`是什么？
2. `cbs.destory`是什么？

```javascript
function invokeDestroyHook (vnode) {
  let i, j
  const data = vnode.data
  if (isDef(data)) {
    if (isDef(i = data.hook) && isDef(i = i.destroy)) i(vnode)
    for (i = 0; i < cbs.destroy.length; ++i) cbs.destroy[i](vnode)
  }
  if (isDef(i = vnode.children)) {
    for (j = 0; j < vnode.children.length; ++j) {
      invokeDestroyHook(vnode.children[j])
    }
  }
}
```

#### 0x06 function invokeInsertHook

`invokeInsertHook`调用插入的钩子，这里会将需要被插入的钩子先存储到`queue[i].data.hook`队列中的钩子等待调用。

如果是初始化状态，并且`vnode`有`parent`属性，则将需要被插入的队列给到`vnode.parent.data.pendingInsert`属性中等待调用。

否则，`initial`非初始化状态，或者没有`vnode.parent`时，则给`queue[i].data.hook`插入`queue[i]`自己，等待hook钩子被调用。

```javascript
function invokeInsertHook (vnode, queue, initial) {
  if (isTrue(initial) && isDef(vnode.parent)) {
    vnode.parent.data.pendingInsert = queue
  } else {
    for (let i = 0; i < queue.length; ++i) {
      queue[i].data.hook.insert(queue[i])
    }
  }
}
```

#### 0x07 function createElm

`createElm`用于创建vnode的元素，这里接收了7个参数：
1. `vnode`虚拟节点
2. `insertedVnodeQueue`等待被插入的虚拟节点队列
3. `parentElm`父级虚拟节点元素
4. `refElm`引用元素
5. `nested`是否嵌套（？）
6. `ownerArray`所有者的数组
7. `index`索引

1、先判断`createComponent`是否有创建组件的动作，如果`vnode.data`存在，并且有`vnode.componentInstace`，则会返回true，就不往下走了。

2、判断`vnode.tag`是否存在，也就是判断vnode是否有标签，有的话则用`nodeOps.createElement`创建元素，并给到`vnode.elm`，并对vnode进行作用域设置`setScope`。

然后创建子元素`createChildren`，如果存在`vnode.data`则调用create钩子`invokeCreateHooks`，然后插入`vnode.elm`元素。

3、如果是注释，创建注释插入。

4、如果是文本，创建文本插入。

```javascript
function createElm(
    vnode,
    insertedVnodeQueue,
    parentElm,
    refElm,
    nested,
    ownerArray,
    index
) {
    // something code...
    if(createComponent(vnode, insertedVnodeQueue, parentElm, refElm)) {
        return
    }
    // something code...
    if(isDef(tag)) {
      vnode.elm = vnode.ns
        ? nodeOps.createElementNS(vnode.ns, tag)
        : nodeOps.createElement(tag, vnode)
      setScope(vnode)
      createChildren(vnode, children, insertedVnodeQueue)
      if (isDef(data)) {
        invokeCreateHooks(vnode, insertedVnodeQueue)
      }
      insert(parentElm, vnode.elm, refElm)
    } else if (isTrue(vnode.isComment)) {
      vnode.elm = nodeOps.createComment(vnode.text)
      insert(parentElm, vnode.elm, refElm)
    } else {
      vnode.elm = nodeOps.createTextNode(vnode.text)
      insert(parentElm, vnode.elm, refElm)
    }
}
```

这个会大量用到，也记一下吧，有ref就插入到ref前面，否则一律插入到最后。

#### 0x08 function insert
```javascript 
function insert (parent, elm, ref) {
  if (isDef(parent)) {
    if (isDef(ref)) {
      if (nodeOps.parentNode(ref) === parent) {
        nodeOps.insertBefore(parent, elm, ref)
      }
    } else {
      nodeOps.appendChild(parent, elm)
    }
  }
}
```

### 0x09 如果传入了oldVnode会发生什么

现在来看看else部分的代码，传入了oldVnode后会发生什么？

1、首先会对元素进行判断，`oldVnode`是否为真元素，`const isRealElement = isDef(oldVnode.nodeType)`。

如果为真，则把`oldVnode`与`vnode`进行对比，如果相同，则开始对虚拟节点进行补丁`patchVnode(oldVnode, vnode, insertedVnodeQueue), null, null removeOnly)`，针对虚拟节点的文本与子虚拟节点进行处理，如果`vnode`有data并且已经被patch过，则调用更新钩子，还有`domdiff`算法也在`patchVnode`函数中，扯的有点远了。

```javascript
const isRealElement = isDef(oldVnode.nodeType)
if (!isRealElement && sameVnode(oldVnode, vnode)) {
    // patch existing root node
    patchVnode(oldVnode, vnode, insertedVnodeQueue, null, null, removeOnly)
} else {
    // something code...
}
```

2、回到刚才，`oldVnode`与`vnode`不相等时，则走else的代码。

如果oldVnode是一个真元素时，则给oldVnode创建一个空元素。

接着缓存oldElm与parentElm两个元素，然后给`vnode`创建一个新的元素，如果给新建的`vnode`有父节点，则递归处理。

然后如果oldElm存在父元素，就删除父元素下的`[oldVnode]`，如果`oldVnode.tag`存在，则调用销毁钩子。

```javascript
const isRealElement = isDef(oldVnode.nodeType)
if (!isRealElement && sameVnode(oldVnode, vnode)) {
    // something code...
} else {
  if (isRealElement) {
    // create an empty node and replace it
    oldVnode = emptyNodeAt(oldVnode)
  }
  
  // replacing existing element
  const oldElm = oldVnode.elm
  const parentElm = nodeOps.parentNode(oldElm)
  
  // create new node
  createElm(
    vnode,
    insertedVnodeQueue,
    // extremely rare edge case: do not insert if old element is in a
    // leaving transition. Only happens when combining transition +
    // keep-alive + HOCs. (#4590)
    oldElm._leaveCb ? null : parentElm,
    nodeOps.nextSibling(oldElm)
  )
  
  // update parent placeholder node element, recursively
  if (isDef(vnode.parent)) {
    // something code...
  }
  
  // destroy old node
  if (isDef(parentElm)) {
    removeVnodes(parentElm, [oldVnode], 0, 0)
  } else if (isDef(oldVnode.tag)) {
    invokeDestroyHook(oldVnode)
  }
}
```

3、回到刚刚的代码，`vnode`如果存在父虚拟节点，这块代码主要时为了更新父元素占位用，虽然我看部太明白，可能核心代码就这一句？`ancestor.elm = vnode.elm`

```javascript
// something code...

// update parent placeholder node element, recursively
if (isDef(vnode.parent)) {
    let ancestor = vnode.parent
    const patchable = isPatchable(vnode)
    while (ancestor) {
      for (let i = 0; i < cbs.destroy.length; ++i) {
        cbs.destroy[i](ancestor)
      }
      ancestor.elm = vnode.elm
      if (patchable) {
        for (let i = 0; i < cbs.create.length; ++i) {
          cbs.create[i](emptyNode, ancestor)
        }
        // #6513
        // invoke insert hooks that may have been merged by create hooks.
        // e.g. for directives that uses the "inserted" hook.
        const insert = ancestor.data.hook.insert
        if (insert.merged) {
          // start at index 1 to avoid re-invoking component mounted hook
          for (let i = 1; i < insert.fns.length; i++) {
            insert.fns[i]()
          }
        }
      } else {
        registerRef(ancestor)
      }
      ancestor = ancestor.parent
    }
}

// something code...
```

#### 0x0A 总结：
patch函数主要就是将vnode转化成真正的dom元素，期间还会有一些oldVnode与vnode的对比，然后在`patchVnode`的虚拟节点进行新增、删除，或者用`updateChildren`做domdiff对比。

如果你给oldVnode的是一个真元素，则会对真元素进行虚拟化，再与vnode进行对比。

因为patch函数是在vdom下的，涉及到视图的处理，代码量必然很多，而且还需要考虑优化节点更换的算法，所以代码量会特别复杂，个人感觉大致看一遍有什么用感觉就可以了。

遗留下的问题：
1. `vnode.data.hook.destory`是什么？
2. `cbs.destory`是什么？
3. `queue[i].data.hook`在哪里用了？