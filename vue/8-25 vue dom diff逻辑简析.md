# vue dom diff逻辑简析
1. 对比前会先拿到新、老元素的startIndex、endIndex，与新老Vnode
2. 开始遍历Vdom，只要新、旧元素的索引都没有撞上，就一直进行遍历
3. 满足新、老元素首节点相同，新、老vnode不变，当前startVnode索引进位，都拿下一个vnode。
4. 满足新、老元素尾节点相同，新、老vnode不变，当前endVnode索引退位，都拿上一个vnode。
5. 满足老首节点、新尾节点相同，老首节点会插入到新尾节点后一位元素节点之前。也就是老首节点会插入到新尾节点的位置。最后oldStartIndex进位，拿下一位oldStartVnode，newEndIndex退一位，拿上一位的newEndVnode。
6. 如果以上6种情况都没有出现，则进入取老vnodeList的key做Map。然后拿当前newStartVnode进行比对。
6. 如果新有、老没有，则用createElm新建，如果新有、老也有，则判断新、老vnode是否相同，**相同**就给老的标记undefined，将相同的老vnode插入到待被处理的vnode之前（oldStartrIndex节点之前）；**不相同**则继续新建vnode插入。
7. 通过以上方法对比完成后，将进入后续补救环节，判断新、老节点索引的问题。
8. **新的开始索引**大于**新的结束索引**，说明新节点列表都比完了，还有老的节点存在，东西肯定多了，需要干掉一些节点，将剩下的老开始、结束节点进行遍历，只要是存在的都干掉。
9. 如果是**老的开始索引**大于**老的结束索引**，就先通过**新结束节点的索引**的下一位索引，拿到的vnode是否undefined？来判断到底要往前插入还是往后插入，一般情况下都是append插入，但是还存在一种情况需要insertBefore，暂时还没确定是什么边界条件。 


##### Q：这个2的替换，到底是替换newCh还是oldCh？

##### A：都不是，替换的是真实的dom

##### Q：dom diff是通过什么进行对比

##### A：dom diff有一个函数叫做sameVnode，它会通过多种形式对vnode进行对比：
1. vnode.key
2. vnode.tag
3. vnode.isComment
4. vnode.data
5. 以上都相等再vnode的inputType，非input默认相等

##### A: 如果key不相等，则判断异步占位等信息，有兴趣可以去源码看看这个函数
1. a.isAsyncPlaceholder === true
2. a.asyncFactory
3. b.asyncFactory.error === undefined | null

##### Q：dom diff算法优点是什么？

##### A：个人认为，没有遍历对比，空间复杂度低。
1. 首先对比头或尾节点是否相等，相等直接略过索引，再对比头和尾、尾和头是否相等，相等也忽略索引。
2. 头头，尾尾，头尾，尾头不相等，则拿到老节点存在的key，做成一个Map，通过新节点去老节点进行匹配，Map结构的优势就是查询速度快，不需要像数组类结构需要便利查询。
3. 如果新Vnode没有命中匹配，则将此直接新增至页面中，如果匹配命中，则将被命中的老Vnode标记undefined。无论此时有无命中匹配，新Vnode的start索引都会进1。
4. 最后再统一对老Vnode进行处理，如果老的索引头大于尾，说明有新增

---
#### newStartVnode.key在oldKeyMap找不到的情况

对于newStartVnode没有在oldKeyMap中找到匹配，需要新建的问题，主要是将内容传入到createElm函数中进行处理：

`createElm函数`会根据传入的parentElm，vnode，refElm进行操作，函数本身会对tag、isComment、Text进行处理。

先对vnode的children传入`createChildren`进行子节点的创建，如果children是一个数组，则对其进行遍历调用createElm，否则判断它如果是原始数据类型，则直接插入。

---

#### newStartVnode.key在oldKeyMap中能找到的情况
如果newStartVnode.key能在oldKeyMap中找到对应的索引，则将该索引从oldVnodeList中拿到oldVnode，打上需要被移动的标识。然后对newStartVnode与需要被移动的标识进行对比。

相同就直接给原来oldVnode标记上undefined，然后将需要被移动的vnode插入到oldStartVnode之前。因为目前newStartVnode是拍的很前，主要是现在已经匹配到了，所以肯定是在oldStartIndex之前，而且oldStartIndex还待匹配，随时会变动，所以挪到它之前准没错。

不相同的话则继续创建，创建完后依然是插入到oldStartVnode的位置之前

---

1. dom diff算法最主要就是用了4个指针两两对比。
2. 满足就往前走，如果4个指针相互对比都没找出想要的东西。
3. 就用空间换时间，让老Vdom以value作为key进行查找。
4. 如果查询成功，则在老vdom设置undefined，新vdom指针进一位。
5. 如果没有查到，则新vdom进一位，直接向页面新增元素。
6. 对比完成后判断新老vdom的指针。
7. 如果老的对比完成，新的还有，则将所有新的都添加过来。
8. 如果新的完成对比，老的还有，则将老的没有进行对比的都干掉。

PS：
1. vue的dom diff是深度优先比较。
2. 老的对比完成，新的还有，说明新的基本都是老的没有的，所以可以直接添加过来。
3. 新的对比完成，老的还有，说明新的改动中没有老的元素，干掉了很正常，没有什么可顾虑的。