看了几天源码，画了思维导图，现在开始用文字整合一下了。

vue渲染的整体逻辑：
1. 先解析拿到的字符串模板代码拿到AST。
2. 处理AST，经过一系列的generate生成字符串，整合到`with(this) { return xxx }`中，这个过程会去处理v-bind、v-model等html中的属性。
3. 生成完毕后会给到当前实例`this.$options.render`属性中，然后才开始挂载当前的组件，也就是$mountComponent。
4. 在组件mount的过程中，会创建一个watcher来观察组件内的值是否有变化，当然创建完成watcher后会顺带执行一次updateComponent，这个过程会收集模板所用到过的属性依赖。
5. 如果有组件，请回到步骤1重新来一遍，解析字符串成AST，再处理AST中的Attrs、Props、Directives等，再进行整合挂载。
6. 综上所述，vue是通过修改属性，再根据属性的dep进行notify，notify后会触发观察这个依赖的watcher进行updateComponent，再执行render的过程。

---

这里我们先不看AST是如何解析的，直接从AST树组合开始。

### 0x00 挂载

在Vue初始化时，处理各种初始化，把双向绑定，事件啥的都解析好，进入挂载阶段。

```javascript
vm.$mount(vm.$options.el);
```

### 0x01 模板编译前置工作

挂载阶段会调用`compileToFunctions`函数，此时将template的字符串进行编译，编译部分有点长，我们一步一步的拆解。

```javascript
var ref = compileToFunctions(template, {
  outputSourceRange: "development" !== 'production',
  shouldDecodeNewlines: shouldDecodeNewlines,
  shouldDecodeNewlinesForHref: shouldDecodeNewlinesForHref,
  delimiters: options.delimiters,
  comments: options.comments
}, this);
```

### 0x02 compileToFunctions，将模板字符串变异成相对应的函数

啥意思呢？大致的理解就是将字符串模板编译成我们常用的`h(tag, attrs, children)`的函数。

这个函数主要的职能是将模板编译成下面这种对象：
```javascript
const compiled = {
  ast: {}, // 解析html标签后的抽象语法树
  render: '_c(tag, attrs, children)', // 通过编译字符串模板后转成的函数字符串
  staticRenderFns: '', 
  errors: [], // 编译器的报错
  tips: [] // 编译器的报错后的提示
}
```

然后将compiled.render与compiled.staticRenderFns进行处理整合到一个叫res的对象并返回出去。

优化部分：如果在编译器的cache中已经有这个key，那么再次编译时就不再对此进行处理了，直接返回cache[key]

```javascript
if(cache[key]) return cache[key]

const res = {}
const fnGenErrors = {}
res.render = createFunction(compiled.render, fnGenErrors)
res.staticRenderFns = compiled.staticRenderFns
   .map(code => createFunction(code, fnGenErrors))

return (cache[key] = res)
```

而刚刚说到的compiled对象，则是通过compile函数来的
```javascript
const compiled = compile(template, options)
```

### 0x03 compile，模板编译

这个函数是由2层函数包装而来的高阶函数，比compileToFunctions还要多一层。


层数 | <div style="width: 150px">方法名</div> | 职能
-|:-|-
1 | createCompilerCreator | 接收baseCompile: Function参数，返回createCompiler函数。
2 | createCompiler | 接收baseOptions: Object，返回compiler函数与compilerToFunctions函数。
3 | compile | 接收2个参数：template与options。用于处理第2层的baseOptions与options的合并，最终将传入的template与合并后的options丢进第一层的baseCompiler中，生成0x02的compiled所需要数据的对象。

通过上面的表格可以看出隔层的作用，我们这里主要是讲第三层。

#### options与baseOptions合并

compile在编译template之前，会先合并baseOptions与options得到finalOptions，主要是对modules、directives进行对象合并，然后再处理其他的属性
```javascript
const finalOptions = Object.create(baseOptions)

if(options.modules) {
  finalOptions.modules = (baseOptions.modules || []).concat(options.modules)  
}
if(options.directives) {
  finalOptions.directives = extend(
    Object.create(baseOptions.directives || {}),
    options.directives
  )
}
for(const key in options) {
  if(key === 'modules' && key === 'directives') continue
  finalOptionsp[key] = options[key]
}
```

extend的实现如下，很简单，就不多说了
```javascript
function extend(to, _from) {
  for(const key in _from) {
    to[key] = _from[key]
  }
  return to
}
```

以上，就是`options编译配置项`的合并。

#### 编译template
编译template部分就是得到我们想要的ast、render、staticRenderFns的对象。

```javascript
var compiled = baseCompile(template.trim(), finalOptions);
```

这段比较段，因为要延伸到下一个函数了。

### 0x04 baseCompiler，基本编译函数
baseCompiler是在初始化第一层createCompilerCreator时传入的，这里做了几个工作：
1. 利用parse函数将string解析成AST。
2. 再利用generate函数将AST组成函数形式执行，生成对应的html模板。
3. 最后返回一个`{ ast, render, staticRenderFns }`对象。

```javascript
function baseCompiler(template, options) {
  const ast = parse(template, options)
  const code = generate(ast, options)
  return {
    ast: ast,
    render: code.render,
    staticRenderFns: code.staticRenderFns
  }
}
```

这里也不难，一目了然，`parse`函数因为是解析抽象语法树，内容个比较庞大这里进行研究，所以下面主要看`generate`函数。

### 0x05 generate，将ast生成我们需要的代码
generate函数的内容也不多，主要还是接收2个参数，ast与options编译配置。而且目的也很单纯，就是为了生成两个东西：`render`与`staticRenderFns`。

```javascript
function generate(ast, options) {
  const state = new CodegenState(options)
  const code = ast ? genElement(ast, state) : '_c("div")'
  return {
    render: (`with(this) { return ${code} }`),
    staticRenderFns: state.staticRenderFns
  }
}
```

这里有两个点，第一个是`CodegenState`，第二个是`genElement`

#### CodegenState

这个是代码生成的状态，这里会将`编译配置options`需要的东西提取到CodegenState的成员属性当中，以备后续genElement要用到。

```javascript
class CodegenState {
  constructor (options: CompilerOptions) {
    this.options = options
    this.warn = options.warn || baseWarn
    this.transforms = pluckModuleFunction(options.modules, 'transformCode')
    this.dataGenFns = pluckModuleFunction(options.modules, 'genData')
    this.directives = extend(extend({}, baseDirectives), options.directives)
    const isReservedTag = options.isReservedTag || no
    this.maybeComponent = (el: ASTElement) => !!el.component || !isReservedTag(el.tag)
    this.onceId = 0
    this.staticRenderFns = []
    this.pre = false
  }
}
```

#### genElement
这个是接下来的重点，生成元素的方法，接下来的内容都会比较枯燥，因为主要都是针对模板的解析逻辑。先上代码再说吧：
```javascript
function genElement(el, state) {
  if(el.parent) {
    el.pre = el.pre || el.parent.pre
  }
  if(el.staticRoot && el.staticProcessed) {
    return genStatic(el, state)
  } else if(el.once && !el.onceProcessed) {
    return genOnce(el, state)
  } else if(el.for && !el.forProcessed) {
    return genFor(el, state)
  } else if(el.if && !el.ifProcessed) {
    return genIf(el, state)
  } else if(el.tag === 'slot') {
    return genSlot(el, state)
  } else {
    // 组件或者元素的情况
    let code
    if(el.component) {
      code = genComponent(el.component, el, state)
    } else {
      let data
      if(!el.plain || (el.pre && state.maybeComponent(el))) {
        data = genData(el, state)
      }
      const children = el.inlineTemplate ? null : genChildren(el, state, true)
      code = `_c(${el.tag}, 
        ${data ? `, ${data}` : ''}, 
        ${children ? `, ${children}` : ''})`
    }
    // 模块转化
    for(let i = 0; i < state.transform.length; i++) {
      code = state.transform[i](el, code)
    }
    return code
  }
}
```

代码主要针对以下几种不同情况进行了处理：
1. astElement有parent时，给astElement对象记录pre属性
2. 有staticRoot属性的，调用生成static方法`genStatic`
3. 有once属性的，调用生成once方法`genOnce`
4. 有for属性的，调用生成for方法`genFor`
5. 有if属性的，调用生成if方法`genIf`
6. 如果el.tag为slot的，调用生成slot方法`genSlot`
7. 如果以上都不是，说明需要生成元素或者组件了，如果ast中有解析到attrs，props之类的属性，那么el.plain就是不平的，然后就进入`genData`，然后针对children进行处理，最后拼成`_c(tag, attrs, children)`的格式返回。

总的来是为了把ast转换成创建标签的函数，并对标签属性进行一系列的处理，最后返回到外部与with结合使用，也就这样：
```javascript
with(this) { return _c(tag, attrs, children)}
```

到这里，genElement的逻辑就完了，剩下就开始揪各种生成器`genXXX`的细节，这些生成器都实现了什么。


---

1-7. 讲完genElement，接下来讲gen的细节


吸收量太大了，今天扛不住，下次再重新debugger吧。

目标：
1. debugger清楚AST是怎么生成
2. 有没对:prop进行处理
3. 如果v-bind:prop的话又会怎么样
4. render时是怎么触发属性的get拦截器
5. 主要目的，将ast生成到的匿名函数与render逻辑结合理解