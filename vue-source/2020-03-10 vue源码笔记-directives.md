### 0x00 genDirectives

凡是标签中带`v-xxx`的在vue的眼里都看做为`指令`，vue把指令分为两种：
1. vue指令，比如v-model、v-if等
2. 自定义指令，directives下配置的函数都属于自定义指令

### 0x01 vue模板解析ast

那么进入正题，凡是标签中带有`v-xxx`，在字符串往抽象语法树解析的过程中，都会被解析成一个对象（~~废话~~），然后存入vnode.directives数组中等待处理。大致情况如下：
```javascript
var vnode = new VNode()
vnode.directives.push({
  name: "model"
  rawName: "v-model"
  value: "text"
  arg: null
  isDynamicArg: false
  modifiers: undefined
  start: 143
  end: 157
})
```

解析成ast部分的详情后续会再详细写，目前只是读了generate函数。

### 0x02 generate函数将ast生成可执行的代码

### 0x03 genElement
解析成ast后就要用generate函数解析，在genElement中会判断当前标签是否平坦，只要是有任何属性，那么一定有`el.plain === false`。于是就进入生成自定义属性的函数`genData(el, state)`了

```javascript
function genElement(el, state) {
  // code...
  
  if(!el.plain) {
    genData(el, state)
  }
  
  // code...
}
```

### 0x04 genData
genData函数中，会首先执行genDirectives判断指令

```javascript
function genData(el, state) {
  var data = '{'
  var dirs = genDirectives(el, state)
  if(dirs) { data += dirs + ',' }
  // code...
}
```

### 0x05 genDirectives
在genDirectives中，会生成`"directives:[.+]"`的字符串返回。核心内容是遍历传入的el.directives，通过el.directives.name拿到state.directives的函数并执行，这个函数下面简称gen。有的gen会返回true保持在运行时调用，保留大部分的属性返回出去。

```javascript
function genDirectives() {
  var dirs = el.directives
  var res = 'directives:['
  var needRuntime
  for(var i = 0, l = dirs.length; i < l; i++) {
    var dir = dirs[i]
    var gen = state.directives[dir.name]
    needRuntime = true
    if(gen) {
      needRuntime = !!gen(el, dir, state.warn)
    }
    // code...
  }
  // code...
}
```

此时的dir.name是model，所以会找到state.directives中的默认指令model函数。

### 0x06 state.directives.model

这个函数分别会对`组件`、`select`、`checkbox`、`radio`、`textarea`、`input`、`非保留标签`进行了不同的处理。以上7种情况，有2种是不需要被写入到directives的，分别是组件、非保留标签。

```javascript
function model(el, model, _warn) {
  // code...
  if (el.component) {
    genComponentModel(el, value, modifiers);
    // component v-model doesn't need extra runtime
    return false
  } else if (tag === 'select') {
    genSelect(el, value, modifiers);
  } else if (tag === 'input' && type === 'checkbox') {
    genCheckboxModel(el, value, modifiers);
  } else if (tag === 'input' && type === 'radio') {
    genRadioModel(el, value, modifiers);
  } else if (tag === 'input' || tag === 'textarea') {
    genDefaultModel(el, value, modifiers);
  } else if (!config.isReservedTag(tag)) {
    genComponentModel(el, value, modifiers);
    // component v-model doesn't need extra runtime
    return false
  }
  // code...
  return true
}
```

在组件genComponentModel会给el.model进行处理

```javascript
function genComponentModel(el, value, modifiers) {
  // code...
  el.model = {
    value: ("(" + value + ")"),
    expression: JSON.stringify(value),
    callback: ("function (" + baseValueExpression + ") {" + assignment + "}")
  };
}
```

而input与textarea标签分别是调用genDefaultModel函数进行处理，这个函数会把el里的内容通过addProp和addHandler分别派送到el.props和el.events数组中，即ast.props和ast.events。

```javascript
function genDefaultModel(el, value, modifiers) {
  // code...
  
  // 给el.props数组添加prop
  addProp(el, 'value', ("(" + value + ")"));
  // 给el.events数组添加事件函数
  addHandler(el, event, code, null, true);
  
  
  // code...
}
```

返回出去后回到genDirectives函数，因为不是组件和保留标签，所以上面的gen函数一定是返回true，所以就进入了el.directives数组的字符串拼接。也就是下面的`res += 'xxxxxx'`

```javascript
function genDirectives(el, state) {
  // code...
  for(var i = 0, l = el.directives; i < l; i++) {
    // code..  .
    if(gen) { needRuntime = !!gen(el, state, state.warn) }
    if(needRuntime) {
      hasRuntime = true
      res += "{name:\"" 
           + (dir.name) 
           + "\",rawName:\"" 
           + (dir.rawName) + "\"" 
           + (dir.value ? (",value:(" 
           + (dir.value) 
           + "),expression:" 
           + (JSON.stringify(dir.value))) : '') 
           + (dir.arg ? (",arg:" 
           + (dir.isDynamicArg ? dir.arg : ("\"" 
           + (dir.arg) + "\""))) : '') 
           + (dir.modifiers ? (",modifiers:" 
           + (JSON.stringify(dir.modifiers))) : '') 
           + "},";
    }
  }
  // code...
}
```

第一次循环得到如下字符串：

```javascript
'directives:[{name:"model",rawName:"v-model",value:(text),expression:"text"},'
```

当循环完毕后再补最后一个数组括号：

```javascript
'directives:[{name:"model",rawName:"v-model",value:(text),expression:"text"}]'
```

处理完addProps与addHandlers，并把内容放到el.props与el.events后，就要开始生成它们了，下面会分别介绍genProps与genHandlers0。

```javascript
// 生成ast的props内容到字符串
if(el.props) {
  data += `domProps:${genProps(el.props)}`
}

// 生成ast的events内容到字符串
if(el.events) {
  data += `${genHandlers(el.events, false)},`
}
```

### 0x07 genProps

domProps分两种，一种是静态staticProps，一种是动态dynamicProps。
```javascript
function genProps(props: Array) {
  let staticProps = '' 
  let dynamicProps = ''
  
  // code...
}
```

然后会对入参`prop in props`数组进行遍历，只要`prop`含有`dynamic`字段就进行动态处理，否则就以静态处理。
```javascript
for(var i = 0, l = props.length; i < l; i++) {
  const prop = props[i]
  // 转义"\2029段落分隔符"，"\2028行分隔符"
  const value = transformSpecialNewlines(props.value)
  if(prop.dynamic) {
    dynamicProps += `${prop.name},${value},`
  } else {
    staticProps += `"${props.name}": ${value}`
  }
}
```

最后，如果有动态属性，就返回含`动态props`的格式，否则就默认返回`静态props`的格式.
```javascript
staticProps = `{${staticProps.slice(0, -1)}}`
if(dynamicProps) {
  return `_d(${staticProps}, [${dynamicProps.slice(0,-1)}])`
} else {
  return staticProps
}
```

最后得到字符串下面右半部分的字符串，返回到函数外与`"domProps: "`字符串进行拼接。

```javascript
'domProps: ' + '{"value":(text),"value":(text)}'
```

### 0x08 genHandlers
生成handlers与props一样，也是分两种，动态和静态。区别于props的是，这里还多了一个nativeOn，可能是用于weex环境的。
```javascript
function genHandlers(events: Object, isNative: boolean) {
  let prefix = isNative ? 'nativeOn:' : 'on:'
  let staticHandlers = ""
  let dynamicHandlers = ""
  // code...
    
}
```

接下来对入参`events对象`进行循环处理：
```javascript
for(var name in events) {
  const handlerCode = genHandler(events[name])
  // code...
}
```

首先会生成handler的代码，所以要去genHandler函数，如果events是数组就会对它进行递归处理。最后都是要处理成`"function($event) { xxx }"`的形式返回

```javascript
function genHandler(handler: Array | Object): string {
  if(Array.isArray(hanlder)){
    return ("[" + (handler.map(function (handler) { return genHandler(handler); }).join(',')) + "]")
  }
  
  // code...
  
  return ("function($event){" + handler.value + "}") 
}
```

再回到genHandlers的for循环：
```javascript
for(var name in events) {
  const handlerCode = genHandler(events[name])
  if(events[name]  && events[name].dynamic) {
    dynamicHandlers += `${name},${handlerCode},`
  } else {
    staticHandlers += `"${name}": ${handlerCode},`
  }
}
```

最后返回前加上prefix字符串进行返回：
```javascript
staticHandlers = "{" + (staticHandlers.slice(0, -1)) + "}";
if (dynamicHandlers) {
  return prefix + "_d(" + staticHandlers + ",[" + (dynamicHandlers.slice(0, -1)) + "])"
} else {
  return prefix + staticHandlers
}
```
到这里，Directives部分已经处理完了，将镜头切到genElement执行完以后的部分：
```javascript
`_c('input',{directives:[{name:"model",rawName:"v-model",value:(text),expression:"text"}],domProps:{"value":(text),"value":(text)},on:{"input":[function($event){if($event.target.composing)return;text=$event.target.value},function($event){if($event.target.composing)return;text=$event.target.value}]}})`
```

格式化后：
```javascript
`_c(
  'input',
  {
    directives:[
      {
        name:"model",
        rawName:"v-model",
        value:(text),expression:"text"
      }
    ],
    domProps:{
      "value":(text),
      "value":(text)
    },
    on:{
      "input":[
        function($event){if($event.target.composing)return;text=$event.target.value},
        function($event){if($event.target.composing)return;text=$event.target.value}
      ]
    }
  }
)`
```

目前generate部分对ast.directives的处理已经完成了，剩下的估计就要到render部分再进行处理。