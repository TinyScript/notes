### 0x0B genSlot

当AST对象中的tag属性，也就是代码中的`el.tag`值为slot时，就会跑genSlot函数了，下面我们来观摩一下genSlot函数

插槽首先是要拿到这个插槽的名字，以及插槽的子元素

```javascript
function genSlot(el, state) {
  const slotName = el.slotName || 'default'
  const children = genChildren(el, state)
  // code...
}
```

拿到slotName与children都是为了生成`render类函数`的字符串，这里说的render类函数，指的就是接下来的`_t()`这一类格式。这里的`_t函数`就是`renderSlot函数`。

```javascript
function genSlot() {
  const slotName = el.slotName || 'default'
  const children = genChildren(el, state)
  let res = `_t('${slotName}', ${ children ? `,${children}` : '' }`
  
  // code...
  
  return res + ')'
}
```

在我debugger的代码中children没有内容，所以res得到的字符串为`_t('default')`，即使有的话，按`_t(slotName, attrs, children)`这个格式拼就是了。

```javascript
`_t('default', { ...attrs }, [ ...children ])`
```

至于renderSlot，内容也不多，主要的职责就是从vm.$scopedSlots找到对应key存放的内容并返回出去，与外部的render类函数进行合并，这个“内容”指的是父级传入给组件的插槽的VNode。

```javascript
function renderSlot(
  name: string,
  fallback: ?Array<VNode>,
  props: ?Object ,
  bindObject: ?Object
) {
  const scopedSlotFn = this.$scopedSlots[name]
  let nodes = scopedSlotFn(props)
  return nodes
}
```

renderSlot主要的逻辑就是渲染父级给的slot内容，`this.$scopedSlots`里的东西就是render函数一开始就执行的normalizeScopedSlots函数。一开始还看不明白，现在总算知道这个是用来联系父级的插槽内容。

以下就是源代码，只要存在父级虚拟节点，就对slots进行处理，此函数的目的就是返回一个value为函数的map，执行函数后会返回父级传进来的vnode。

```javascript
if (_parentVnode) {
  vm.$scopedSlots = normalizeScopedSlots(
    _parentVnode.data.scopedSlots,
    vm.$slots,
    vm.$scopedSlots
  )
}
```

vue的render最终目的就是将generate生成的“render类函数”执行，而“render类函数”执行后会返回一个vnode tree，也就是我们常说的虚拟dom，所以genSlot生成的`_t函数 => renderSlot`也不例外，根据`el.slotName`从`vm.$scopeSlots`找到对应的vnode并返回。

#### genSlots总结

genSlot不会在首次组件挂载时触发，因为`<slot>`必须要卸载组件里面。

而组件的创建时间，正是要在下一次updateComponent时，执行Vue._update后，进入patch函数创建真实元素时调用createElm函数，才会轮到组件的创建。

组件创建后就要进入$mount挂载阶段，挂载前需要对组件进行ast解析编译，才有机会调用generate函数生成代码，然后调用genElement创建元素，才有机会到genSlot这个函数。

总结：在updateComponent时，VNode在`patch的阶段`子组件才会被创建，子组件被创建的过程，才有机会执行genSlot函数。

在父级会经过genElement的genData函数会将传入的slot解析成如下代码：
```javascript
`_c(
  'demo1', 
  [
    _c(
      'div', 
      { 
        attrs: { 'slot': 'content' }, 
        slot:'content', 
      }
    )
  ]
)`
```

然后到了genSlot又会解析成如下：

```javascript
`_t('content', { ...attrs }, [ ...children ])`
```

### 0x0C genChildren

这个函数在之前各个genXXX函数都有涉及就不细讲了，大家也熟悉，我就直接讲总结了。

genChildren目的就是为了把ASTNode处理成字符串，并传到`render类函数`子节点部分的代码。

也就是Array<ASTNode> => string转化的一个过程，比如：

```javascript
function genChildren(children) {}
genChildren([ASTNode, ASTNode, ASTNode])
// => '[_c("tagName",{...attrs}, {...children}),_c("tagName"), _c("tagName")]'

// 最终是要放在children数组部分
/**
 * => _c(
 *      'div', 
 *      {...attrs}, 
 *      [
 *        _c("tagName",{...attrs}, {...children}),
 *        _c("tagName"), 
 *        _c("tagName")
 *      ]
 *    )
 */
```

### 0x0D 结尾 2020.03.16
好嘞，render前置部分已经讲完了，render过程因为debugger时过了一遍，就顺手也粗略的写了一下笔记，没有细写，render部分与update阶段一起写笔记。

接下来先对字符串模板解析成抽象语法树部分进行解读。