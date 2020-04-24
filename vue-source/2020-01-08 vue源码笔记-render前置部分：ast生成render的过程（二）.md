> 之前讲到genElement，现在我们就开始细讲genElement内使用过的方法。

### 0x06 genElement内使用过的方法

1. genStatic -- 处理v-pre用
2. genOnce
3. genFor
4. genIf
5. genChildren
6. genSlot
7. genComponent
8. genData

然后我们在这里一个一个的分析，在开始分析之前，先罗列一下ast转换需要用到的一些方法简称：
```javascript
vm._c = (a, b, c, d) => createElement(vm, a, b, c, d, false)

function installRenderHelpers (target: any) {
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
}
```

### 0x07 genStatic

debugger的时候，发现似乎带有v-pre的子节点，才会进入这个方法，用来生成静态节点用存到`staticRenderFns`，最后会生成`"_m(index)"`的字符串，而`_m`对应的函数就是`renderStatic`。最后会在render方法中的字符串调用`_m`函数，从而到了renderStatic函数，遍历调用staticRenderFns方法。

```javascript
function genStatic(el: ASTElement, state: CodegenState): string {
  el.staticProcessed = true
  state.staticRenderFns.push(`with(this) { return ${genElement(el, state)} }`)  
  return `_m(${
    state.staticRenderFns.length - 1
  }${
    el.staticInFor ? ',true' : ''
  })`
}
```

首先看这个函数的出入参，就明白这肯定是要处理ASTElement对象的与static相关的属性，然后返回一个字符串出去，所以我们目的就是要生成一个string出去，再顺便处理一下需要处理的东西。

开始看正文，开始的时候将当前的对象标记为已被static处理。
```javascript
el.staticProcessed = true
```
然后再对`state.staticRenderFns`方法进行填充，因为是静态渲染方法，所以就填充一个与外面正常渲染相同的render属性字符串。
```javascript
{  
  render: `with(this) { return ${genElement(el, state)} }`
}
```
最后返回字符串`_m`，一开说过了，就不细说。


去文档看了一下pre相关的指令，主要的职能是为了跳过编译，说白了就是一个静态的节点，不会被当做额外的依赖收集。
```html
<!-- 这段代码不会被编译，包括子节点 -->
<span v-pre>{{ this will not be compiled }}</span>
```

### 0x08 genOnce
这个方法代表只渲染一次，后续将作为静态节点不进行处理，对应的当然就是`v-once`这个指令了。

```javascript
function genOnce(el: ASTElement, state: CodegenState): string {
  el.onceProcessed = true
  if(el.if && el.!ifProcessed) {
    return genIf(el, state)
  } else if(el.staticInFor) {
    let key = ''
    let parent = el.parent
    while(parent) {
      if(parent.for) {
        key = parent.key
        break
      }
      parent = parent.parent
    }
    if(!key) {
      return genElement(el, state)
    }
    return `_o(${genElement(el,state)},${state.onceId++},${key})`
  } else {
    return genStatic(el, state)
  }
  
}
```

针对单纯的v-once来说就是else部分，往外丢一个genStatic函数作为静态渲染。既然这样为什么不直接用genStatic函数，还得专门做一个v-once函数呢？

因为与普通的genStatic不一样，而且还要处理if和for时需要用到的单次渲染优化。所以这里专门抽离了一个once的函数来进行不同情况的区分调用，简而言之这个函数是一个`调度者`，而不是一个生产者。

回归正题，genOnce的genStatic有什么不一样呢？唯一不同的地方应该就是在`genElement`->`genChildren`->`gen`->`genNode`->`genText`时的处理了，因为once的ast中有`expression`这个表达式在，即我们看到的`text.express`。
```javascript
function getText(text: ASTText | ASTExpression): text {
  return `_v(${text.type === 2 
    ? text.expression
    : transformSpecialNewlines(JSON.stringify(text.text))
  })`
}
```
所以它实际套的也是`_m`的函数，这个函数处理静态元素的方法是利用了缓存，只要是有_m(index)的，生成过一次就直接从缓存catched里拿，不再进行计算了。

由上面的解释可以说明即使是双向绑定，也是通过对比元素有更新后再通过调用各种`_c('tag')`，`_m(index)`啥的方法进行刷新视图，如果要实现单次渲染，就针对_m传入的索引进行返回。

#### 0x0800 _m函数，即renderStatic函数

```javascript
function renderStatic(
  index: number,
  isInFor: boolean
): VNode | Array<VNode> {
  const vm: Component = this
  const catched = vm._staticTrees || (vm._staticTrees = [])
  let tree = catched[index]
  if(tree && !isInFor) return tree
  
  tree = catched[index] = vm.$options.staticRenderFns[index].call(
    this._renderProxy,
    null,
    this
  )
  markStatic(tree, ("__static__" + index), false);
  return tree
}
```

如果在缓存中有这个索引，用缓存，否则执行一次这个渲染函数（genElement），最后生成的是一个vnode，而text已经被成功解析成字符串。很简单，就不多做解释了。

### 0x09 genFor

先来一段演示，告诉我们genFor要干什么
```html
<div v-for="(item, index, instance) in arr" :key="index">
  {{ item.a }}{{ item.b }}{{ item.c }}
</div>
<script>
export default {
  data() {
    return {
        arr: [
          {a:1,b:2,c:3},
          {a:4,b:5,c:6},
          {a:7,b:8,c:9}
        ]
    }
  }
}
</script>
```
```javascript
// genElement会得到这个
"_c('div',{key:index},[_v(_s(item.a)+_s(item.b)+_s(item.c))])"

// genFor会得到这个
"`_l((arr),function(item,index){return _c('div',{key:index},[_v(_s(item.a)+_s(item.b)+_s(item.c))])})"

// => 格式化一下
`_l(
  (arr),
  function(item,index){
    return _c(
      'div',
      {key:index},
      [
        _v(
          _s(item.a) + _s(item.b) + _s(item.c)
        )
      ]
    )
  }
)`
```

似乎有点不知所以然，我们先来看看ast如何生成for的函数：genFor。

#### 0x0900 genFor函数中干了什么

```javascript
function genFor(
  el: any,
  state: CodegenState,
  altGen: Function,
  altHelper: string
) {
  const exp = el.for // 引用需要遍历的数组变量名
  const alias = el.alias // 遍历的别名，就是引用arr[i]的变量名
  // 索引，如果v-for是一个对象，就是key
  const iterator1 = el.iterator1 ? `,${el.iterator1}` : '' 
  // 空，如果v-for是一个对象，iterator2就是代表索引
  const iterator2 = el.iterator2 ? `,${el.iterator2}` : ''
  
  el.forProcessed = true
  
  return `${altHelper || '_l'}((${exp}),` +
    `function(${alias}${iterator1}${iterator2}) {` + 
      `return ${ (altGen || genElement)(el, state) }`
    `})`
}
```

前5行很简单，就不解释了

从return开始说起吧，首先它会执行一个函数`_l(expression, function() {})`，而这个`_l`函数的全称叫`renderList`，顾名思义渲染list用的，也就是针对v-for处理的函数。

接下来看参数，`expression`就是我们需要被遍历的`数组变量名`。而第二个参数传的是function，这个function有三个参数，这三个参数上边都有注释，不解释。根据下面的html代码，三个参数最终会被解析为`item, index, instance`

```html
<div v-for="(item, index, instance) in arr"></div>
```

但是现在有个问题，这匿名函数的三个参数在函数里也没被用到啊？那解析来有啥用。

疑问在这里解释，现在看这个匿名函数内部执行了`genElement`，这个就是解释问题关键点了，因为el.for如果有东西的话，就表明要生成一个for专用的字符串模板。这里就分别使用到了`item`和`index`也就是上面写过的：
```javascript
"_c('div',{key:index},[ _v(_s(item.a)+_s(item.b)+_s(item.c)) ])"
```

上面的字符串代码是由genFor函数里调用genElement函数得到的。

```javascript
function genFor(el, state, altGen, altHelper) {
  // code ...
  return (altHelper || '_l') + "((" + exp + ")," +
    "function(" + alias + iterator1 + iterator2 + "){" +
      "return " + ((altGen || genElement)(el, state)) +
    '})'
}
```

#### 0x0901 genFor函数的调用链

genElement会调用genChildren。

```javascript
function genElement(el, state) {
  // code...
  
  var children = el.inlineTemplate ? null : genChildren(el, state, true);
  
  // code..
}
```

genChildren会通过map的方式调用genNode。

```javascript
function genChildren() {
  // code...
  var gen = altGenNode || genNode;
  return `[${children.map(c => gen(c, state)).join(',')}]${
    normalizationType ? `,${normalizationType}` : ''
  }`
}
```

genNode会把所有子元素生成元素、注释或者文本节点。

> ※ 这里开始进入children.map的循环模式

```javascript
function genNode (node: ASTNode, state: CodegenState): string {
  if (node.type === 1) {
    return genElement(node, state)
  } else if (node.type === 3 && node.isComment) {
    return genComment(node)
  } else {
    return genText(node)
  }
}
```

这里有3个分支：
1. genElement的话，就回到genELement函数进行循环
2. genComment的话最终会返回一个`_e`函数字符串，对应一开始的`createEmptyVNode`
3. genText的话最终会返回一个`_v`函数字符串，对应一开始的`createTextVNode`

如果`node.type === 2`的情况下肯定就走genText了，所以自然就会拿到node.express进行填充，最后得到的是`"_v(node.express)"`字符串。

```javascript
function getText(text: ASTText | ASTExpression): string {
    return `_v(${text.type === 2 ? text.expression : transformSpecialNewlines(JSON.stringify(text.text))})`
}
```

> ※ children.map单词循环结束

循环结束后，得到所有根据astNode.children生成的`可被函数执行`的字符串。根据之前的代码，children.map最终会生成这样的代码，_s是toString。

```javascript
genChildren(el, state, true)
// 执行上面的函数，得到下面的数组内容：
`[_v("arr "+_s(instance)+" "+_s(item.a)+_s(item.b)+_s(item.c))]`
```

需要注意的是，这里只是for的单次渲染结束，genChildren最终要返回到genElement进行字符串合并。

```javascript
function genElement(el: ASTElement, state: CodegenState) {
  // code...
  var children = genChildren(el, state, true)
  code = `_c('${el.tag}'${
    data ? `,${data}` : '' // data
  }${
    children ? `,${children}` : '' // children
  })`
  // code...
  return code
}
```

`_c`在上边的词典没有，先说说主要职能吧，它就是createElement函数，主要是用来判断生成`VNode`、`createComponent`、`createEmptyVNode`，本质上这个函数最终返回的都是一个`VNode`，或者是一个`Array<VNode>`。具体的函数如下

```javascript
function createElement(
  context: Component,
  tag: any,
  data: any,
  children: any,
  normalizationType: any,
  alwaysNormalize: boolean
): VNode | Array<VNode> {
  // code...
}

vm._c = (a,b,c,d => createElement(vm, a, b, c, d, false))
```

但是目前只讲generate部分，所以不详细说明。


在genElement函数执行完毕后，就要把代码上浮至genFor函数，根据之前的代码，这里的genElement最红得到的是这样一个字符串。
```javascript
`_c(
  'div',
  {key:index},
  [_v('arr '+_s(instance)+' '+_s(item.a)+_s(item.b)+_s(item.c))]
)`
```

现在上浮到genFor函数，得到最初我们看到的`_l函数`字符串：

```javascript
function genFor() {
  // code...
  return `_l(
    (arr),
    function(item,index){
      return _c(
        'div',
        {key:index},
        [
          _v(
            'arr '+_s(instance)+' '+_s(item.a) + _s(item.b) + _s(item.c)
          )
        ]
      )
    }
  )`   
}

```

至此，genFor的调用链执行完毕，但是有一个东西没交代，啥是`_l函数`？

#### 0x0902 _l 函数，即renderList函数
到了genFor函数，那就必须看看renderList函数。这里的循环实现主要就是依靠_l来进行处理。

renderList函数函数会接收2个参数，val和render。参数1是遍历的目标，参数2是渲染内容的函数。渲染内容函数又分别接收3个参数：参数1是当前遍历内容的变量别名；参数2是索引，如果遍历目标是一个对象，那么它就是一个key；参数3也是索引，只有遍历目标是对象才会有值。

```javascript
function renderList(
  val: any, 
  render: (
    val: any, 
    keyOrIndex: string|number, 
    index: number
  ) => VNode
): ?Array<VNode> {
  // code...
}

const _l = renderList
```

函数内部针对val的类型做了三种处理，分别是`number`、`Array | string`、`Object`

这个函数看上去代码非常多判断，但实际上核心的就只有下面几句：
```javascript
function renderList(val, render) {
  let ret;
  ret = new Array(val.length)
  for(var i = 0; i < val.length; i++) {
    ret[i] = render(val[i], i)
  }
  ret._isVList = true
  return ret
}
```

最终目的就是为了遍历val，然后把它的值传进render函数调用。number、string、Array几个类型操作几乎一样，但是Object类型就有Symbol和普通对象的区别：

```javascript
if (hasSymbol && val[Symbol.iterator]) {
  ret = []
  const iterator: Iterator<any> = val[Symbol.iterator]()
  let result = iterator.next()
  while (!result.done) {
    ret.push(render(result.value, ret.length))
    result = iterator.next()
  }
} else {
  keys = Object.keys(val)
  ret = new Array(keys.length)
  for (i = 0, l = keys.length; i < l; i++) {
    key = keys[i]
    ret[i] = render(val[key], key, i)
  }
}
```

针对Symbol迭代器进行遍历，如果是普通对象，则与数组无区别。

下面结合模板粗略说一下renderList干了啥，模板解析的过程中已经把`item`、`index`和`[1,2,3,4,5]`解析好备用，到最后通过字符串拼接到renderList函数里，然后应该都能看懂了。

```html
<template>
  <div v-for="(item, index) in [1,2,3,4,5]">
    {{ item }} {{ index }}
  </div>
</template>

<script>
function renderList(val, render) {
  let ret;
  ret = new Array(val.length)
  for(var i = 0; i < val.length; i++) {
    ret[i] = render(val[i], i)
  }
  ret._isVList = true
  return ret
}

renderList([1,2,3,4,5], (item, index) => {
  return `_v(${_s(item)} ${_s(index)})`
})
</script>
```

genFor因为嵌套的比较深，写了300行终于结束了。

### 0x0A genIf

前言：在generate->genElement的过程中，如果提供的ASTNode包含if属性，即`el.if`有东西时，就会进入genIf的逻辑，目的是为了生成处理v-if的字符串。

这个函数比较简洁，只有一个设置已处理过if属性的逻辑，然后执行genIfConditions函数并返回出去，看名字的意思像是生成if条件的处理函数。主要是用来处理当前AST节点的ifConditions的内容。

```javascript
function genIf(
  el: any,
  state: CodegenState,
  altGen?: Function,
  altEmpty?: string
): string {
  el.ifProcessed = true
  return genIfConditions(el.ifConditions.slice(), state, altGen, altEmpty)
}
```

#### 0x0A00 genIfConditions生成条件函数

函数接收4个参数，1.条件、2.生成状态、3.自定义生成函数、4.自定义生成空函数。最后会返回一个三元-多元表达式，有几个元就看你的模板有几个else-if了。

```javascript
function genIfConditions(
  conditions: ASTIfConditions,
  state: CodegenState,
  altGen?: Function,
  altEmpty?: string
): string {
  // code...
}
```

代码首先对conditions为空的情况下进行返回一个空元素。

```javascript
if (!conditions.length) {
  return altEmpty || '_e()'
}
```

再对`conditions`数组进行shift取值，拿到的第一个对象`condition.exp`属性进行判断。这里的判断主要是为了处理是`if`、`else-if`还是`else`的逻辑，只有if和else-if才有exp属性，else的exp属性是undefined。

所以只要有if与else-if，就会一直执行genIfConditions，三元表达式就会一直嵌套，最后返回一个三元/多元表达式的字符串给到genElement函数。genTernaryExp函数下面会继续讲

```javascript
const condition = conditions.shift()
if (condition.exp) {
  return `(${condition.exp})?${
    genTernaryExp(condition.block)
  }:${
    genIfConditions(conditions, state, altGen, altEmpty)
  }`
} else {
  return `${genTernaryExp(condition.block)}`
}
```

condition.exp指的就是下面的show变量：

```html
<template>
  <div v-if="show"></div>
  <div v-else></div>
</template>

<script>
  export default {
    data() {
      return {
        show: true
      }
    }
  }
</script>
```

#### 0x0A01 genTernaryExp

现在来看看genTernaryExp函数，看名字的意思是生成三元表达式，但是我还没理解清楚到底是生成三元表达式需要内容，还是概括这个函数内部要实现的东西。

但是代码的意思比较简单，如有传入自己控制生成的函数`altGen`就直接调用altGen，否则就按默认生成VNode的string。默认生成还要区分当前的ast是否有once属性，有的话直接调用genOnce，只执行一次genElement后把内容丢进catched，否则每次都要调用genElement。

```javascript
function genTernaryExp (el) {
  return altGen
    ? altGen(el, state)
    : el.once
      ? genOnce(el, state)
      : genElement(el, state)
}
```

最终我们看到的情况大致如下：

```html
<template>
  <div v-if="show">1</div>
  <div v-else>2</div>
</template>

<script>
// 没有el.once时
function genIf(el, state, altGen, altEmpty) {
  el.processed = true
  return `(${el.conditions[0].exp}) 
         ? ${genElement(el.conditions[0].block, state)}
         : ${genElement(el.conditions[1].block, state)}`
}
// 最终输出 '(show) ? _c("div", [1]) : _c("div", [2])'

// 有el.once时
function genIf(el, state, altGen, altEmpty) {
  el.processed = true
  console.log(el.conditions[0].once) // => true
  return `(${el.conditions[0].exp}) 
         ? ${genOnce(el.conditions[0].block, state)}
         : ${genElement(el.conditions[1].block, state)}`
}
// 最终输出 '(show) ? _m(0) : _c("div")'
</script>
```

这里的block代表当前匹配的exp属性对应的代码块，或者说是ifConditions的exp对应的ast节点。

到这里genIf就已经完结了。

先放出最终生成的字符串代码观摩一下：

```html
<template>
<div>
  <div @click="push">push</div>
  <div @click="click">click</div>
  <div @click="add">add</div>
  <input v-model="text">
  <span v-once>
    result: {{ result }}
  </span>
  <keep-alive>
    <demo1 v-if="show"></demo1>
    <demo2 v-else></demo2>
  </keep-alive>
</div>
</template>
<script>
`[
  _c('div',{on:{"click":push}},[_v("push")]),
  _v(" "),
  _c('div',{on:{"click":click}},[_v("click")]),
  _v(" "),
  _c('div',{on:{"click":add}},[_v("add")]),
  _v(" "),
  _c(
    'input',
    {
      directives:[{name:"model",rawName:"v-model",value:(text),expression:"text"}],
      domProps:{"value":(text)},
      on:{
        "input":function($event){if($event.target.composing)return;text=$event.target.value}
      }
    }
  ),
  _v(" "),
  _m(0),
  _v(" "),
  _c('keep-alive',[(show)?_c('demo1'):_c('demo2')],1)
],1`
</script>
```