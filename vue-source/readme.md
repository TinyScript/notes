# Vue源码走马观花系列笔记

### 初始化mixin
1. [9-20 vue源码笔记-init.js解读笔记](https://github.com/TinyScript/notes/blob/master/vue-source/9-20%20vue%E6%BA%90%E7%A0%81init.js%E8%A7%A3%E8%AF%BB%E7%AC%94%E8%AE%B0.md)
2. [9-25 vue源码笔记-stateMixin解读笔记](https://github.com/TinyScript/notes/blob/master/vue-source/9-25%20vue%E6%BA%90%E7%A0%81stateMixin%E8%A7%A3%E8%AF%BB%E7%AC%94%E8%AE%B0.md)
3. [9-26 vue源码笔记-eventsMixin解读笔记](https://github.com/TinyScript/notes/blob/master/vue-source/9-26%20vue%E6%BA%90%E7%A0%81eventsMixin%E8%A7%A3%E8%AF%BB%E7%AC%94%E8%AE%B0.md)
4. [9-29 vue源码笔记-lifecycleMixin解读笔记](https://github.com/TinyScript/notes/blob/master/vue-source/9-29%20vue%E6%BA%90%E7%A0%81lifecycleMixin%E8%A7%A3%E8%AF%BB%E7%AC%94%E8%AE%B0.md)
5. [10-08 vue源码笔记-renderMixin解读笔记](https://github.com/TinyScript/notes/blob/master/vue-source/10-08%20vue%E6%BA%90%E7%A0%81renderMixin%E8%A7%A3%E8%AF%BB%E7%AC%94%E8%AE%B0.md)

### Dep、Watcher、computed
1. [10-17 vue源码笔记-Dep.js解读笔记](https://github.com/TinyScript/notes/blob/master/vue-source/10-17%20vue%E6%BA%90%E7%A0%81Dep.js%E8%A7%A3%E8%AF%BB%E7%AC%94%E8%AE%B0.md)
2. [10-17 vue源码笔记-Watcher.js解读笔记](https://github.com/TinyScript/notes/blob/master/vue-source/10-17%20vue%E6%BA%90%E7%A0%81Watcher.js%E8%A7%A3%E8%AF%BB%E7%AC%94%E8%AE%B0.md)
3. [11-18 vue源码笔记-new Watcher创建时机笔记](https://github.com/TinyScript/notes/blob/master/vue-source/11-18%20vue%E6%BA%90%E7%A0%81new%20Watcher%E5%88%9B%E5%BB%BA%E6%97%B6%E6%9C%BA%E7%AC%94%E8%AE%B0.md)
4. [11-19 vue源码笔记-依赖收集解读笔记](https://github.com/TinyScript/notes/blob/master/vue-source/11-19%20vue%E6%BA%90%E7%A0%81%E4%BE%9D%E8%B5%96%E6%94%B6%E9%9B%86%E8%A7%A3%E8%AF%BB%E7%AC%94%E8%AE%B0.md)
5. [11-29 vue源码笔记--执行过程笔记](https://github.com/TinyScript/notes/blob/master/vue-source/11-29%20vue%E6%BA%90%E7%A0%81-%E6%89%A7%E8%A1%8C%E8%BF%87%E7%A8%8B%E7%AC%94%E8%AE%B0.md)
6. [12-04 vue源码笔记-Dep.target什么时候会有东西](https://github.com/TinyScript/notes/blob/master/vue-source/12-04%20vue%E6%BA%90%E7%A0%81%E7%AC%94%E8%AE%B0-Dep.target%E4%BB%80%E4%B9%88%E6%97%B6%E5%80%99%E4%BC%9A%E6%9C%89%E4%B8%9C%E8%A5%BF.md)
7. [12-04 vue源码笔记-什么是Dep.target](https://github.com/TinyScript/notes/blob/master/vue-source/12-04%20vue%E6%BA%90%E7%A0%81%E7%AC%94%E8%AE%B0-%E4%BB%80%E4%B9%88%E6%98%AFDep.target.md)
8. [12-16 vue源码笔记-watcher收集dep实例的时机](https://github.com/TinyScript/notes/blob/master/vue-source/12-16%20vue%E6%BA%90%E7%A0%81%E7%AC%94%E8%AE%B0-watcher%E6%94%B6%E9%9B%86dep%E5%AE%9E%E4%BE%8B%E7%9A%84%E6%97%B6%E6%9C%BA.md)
9. [12-16 vue源码笔记-为什么要收集依赖？](https://github.com/TinyScript/notes/blob/master/vue-source/12-16%20vue%E6%BA%90%E7%A0%81%E7%AC%94%E8%AE%B0-%E4%B8%BA%E4%BB%80%E4%B9%88%E8%A6%81%E6%94%B6%E9%9B%86%E4%BE%9D%E8%B5%96%EF%BC%9F.md)
10. [2020-02-29 组件、$watch、computed的watcher逻辑记录](https://github.com/TinyScript/notes/blob/master/vue-source/2020-02-28%20vue%E6%BA%90%E7%A0%81%E7%AC%94%E8%AE%B0-%E6%80%BB%E7%BB%93%EF%BC%9A%E7%BB%84%E4%BB%B6%E3%80%81%24watch%E3%80%81computed%E7%9A%84watcher%E5%85%B3%E7%B3%BB.md)
11. [2020-02-29 computed什么时候才会被调用的逻辑记录](https://github.com/TinyScript/notes/blob/master/vue-source/2020-02-28%20vue%E6%BA%90%E7%A0%81%E7%AC%94%E8%AE%B0-%E8%AE%A1%E7%AE%97%E5%B1%9E%E6%80%A7%E5%87%BD%E6%95%B0%E4%BB%80%E4%B9%88%E6%97%B6%E5%80%99%E6%89%8D%E4%BC%9A%E8%A2%AB%E8%B0%83%E7%94%A8.md)

### 编译AST
1. [2020-03-16 vue源码笔记-字符串解析成AST](https://github.com/TinyScript/notes/blob/master/vue-source/2020-03-16%20vue%E6%BA%90%E7%A0%81%E7%AC%94%E8%AE%B0-%E5%AD%97%E7%AC%A6%E4%B8%B2%E8%A7%A3%E6%9E%90%E6%88%90AST.md)
2. html标签属性解析成AST

### AST -> generate
1. [2019-12-29 vue源码笔记-render前置部分：ast生成render的过程（一）](https://github.com/TinyScript/notes/blob/master/vue-source/2019-12-29%20vue%E6%BA%90%E7%A0%81%E7%AC%94%E8%AE%B0-render%E5%89%8D%E7%BD%AE%E9%83%A8%E5%88%86%EF%BC%9Aast%E7%94%9F%E6%88%90render%E7%9A%84%E8%BF%87%E7%A8%8B%EF%BC%88%E4%B8%80%EF%BC%89.md)
2. [2020-01-08 vue源码笔记-render前置部分：ast生成render的过程（二）](https://github.com/TinyScript/notes/blob/master/vue-source/2020-01-08%20vue%E6%BA%90%E7%A0%81%E7%AC%94%E8%AE%B0-render%E5%89%8D%E7%BD%AE%E9%83%A8%E5%88%86%EF%BC%9Aast%E7%94%9F%E6%88%90render%E7%9A%84%E8%BF%87%E7%A8%8B%EF%BC%88%E4%BA%8C%EF%BC%89.md)
3. [2020-03-10 vue源码笔记-directives](https://github.com/TinyScript/notes/blob/master/vue-source/2020-03-10%20vue%E6%BA%90%E7%A0%81%E7%AC%94%E8%AE%B0-directives.md)
4. [2020-03-12 vue源码笔记-render前置部分：ast生成render的过程（三）](https://github.com/TinyScript/notes/blob/master/vue-source/2020-03-12%20vue%E6%BA%90%E7%A0%81%E7%AC%94%E8%AE%B0-render%E5%89%8D%E7%BD%AE%E9%83%A8%E5%88%86%EF%BC%9Aast%E7%94%9F%E6%88%90render%E7%9A%84%E8%BF%87%E7%A8%8B%EF%BC%88%E4%B8%89%EF%BC%89.md)

### vdom
1. [8-25 vue源码笔记-dom-diff逻辑简析](https://github.com/TinyScript/notes/blob/master/vue-source/8-25%20vue%20dom%20diff%E9%80%BB%E8%BE%91%E7%AE%80%E6%9E%90.md)
2. [10-09 vue源码笔记-vm.__patch__解读笔记](https://github.com/TinyScript/notes/blob/master/vue-source/10-09%20vue%E6%BA%90%E7%A0%81vm.__patch__%E8%A7%A3%E8%AF%BB%E7%AC%94%E8%AE%B0.md)
3. virtual dom生成笔记

### 组件挂载
1. [2020-03-04 vue源码笔记-组件创建的过程](https://github.com/TinyScript/notes/blob/master/vue-source/2020-03-04%20vue%E6%BA%90%E7%A0%81%E7%AC%94%E8%AE%B0-%E7%BB%84%E4%BB%B6%E5%88%9B%E5%BB%BA%E7%9A%84%E8%BF%87%E7%A8%8B.md)

### 忘记了与哪个笔记相关的图
1. [2019-12-16 触发notify的执行路径](https://github.com/TinyScript/notes/blob/master/vue-source/img/%E8%A7%A6%E5%8F%91notify%E7%9A%84%E6%89%A7%E8%A1%8C%E8%B7%AF%E5%BE%84.md)
2. [2019-11-11 追溯computed初始化与触发的过程](https://github.com/TinyScript/notes/blob/master/vue-source/img/%E8%BF%BD%E6%BA%AFcomputed%E5%88%9D%E5%A7%8B%E5%8C%96%E4%B8%8E%E8%A7%A6%E5%8F%91%E7%9A%84%E8%BF%87%E7%A8%8B.md)
3. [2019-12-17 追溯data初始化与update的过程](https://github.com/TinyScript/notes/blob/master/vue-source/img/%E8%A7%A6%E5%8F%91notify%E7%9A%84%E6%89%A7%E8%A1%8C%E8%B7%AF%E5%BE%84.md)
4. [2020-02-25 vue-模板编译函数的关系](https://github.com/TinyScript/notes/blob/master/vue-source/img/vue-%E6%A8%A1%E6%9D%BF%E7%BC%96%E8%AF%91%E5%87%BD%E6%95%B0%E7%9A%84%E5%85%B3%E7%B3%BB.md)
5. [2019-12-19 watcher的依赖收集](https://github.com/TinyScript/notes/blob/master/vue-source/img/watcher%E7%9A%84%E4%BE%9D%E8%B5%96%E6%94%B6%E9%9B%86.md)

### 学习总结
1. [2020-02-29 vue源码学习第一次总结](https://github.com/TinyScript/notes/blob/master/vue-source/2020-02-29%20vue%E6%BA%90%E7%A0%81%E5%AD%A6%E4%B9%A0%E7%AC%AC%E4%B8%80%E6%AC%A1%E6%80%BB%E7%BB%93.md)