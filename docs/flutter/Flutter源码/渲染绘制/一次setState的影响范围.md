# 一次setState的影响范围
在State对象的setState被调用后，会发生三件比较重要的事情：
1. 相关联的Element元素被标记为脏元素(_dirty:true)。
2. 相关联的Element元素被添加到BuildOwner的脏元素表(_dirtyElements)。
3. 检查是否有Vsync信号回调，如果没有注册一个。

在下一次Vsync信号到来时，触发注册的回调，进入BuildOwner的buildScope流程。重点流程在于取出_dirtyElements中的元素，并按顺序执行rebuild。那么我们已StatefulElement元素的rebuild为入口。

1. StatefullElement => rebuild()
2. StatefulElement => performRebuild()
