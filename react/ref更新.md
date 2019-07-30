# ref更新
ref在vue中也有，类似id选择器，但是此次更新了以后，在下一次版本将会剔除字符串形式的ref标记，也就是禁止`ref="xxx"`，随之而来的是`React.createRef()`

`React.forwardRef`会创建一个React组件，并且将ref的属性转发到组件树下面的另一个组件中，主要作用于：
- 转发refs到DOM中
- 在高阶组件转发refs

这么简单，就不贴代码了。