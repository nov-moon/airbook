# StatefulWidget-State 生命周期

## 实例化对象
framework在把StatefulWidget插入到tree中时调用StatefulWidget.createState方法创建State实例，生成的实例会被绑定到StatefulElement，这种绑定是永久的。

* framework何时把StatefulWidget插入到tree？
* 绑定的StatefulElement与StatefulWidget有什么关系？

StatefulElement集成关系：
```
    Element
    ---ComponentElement
        ---StatelessElement
        ---StatefulElement
```
先看下Element、ComponentElement、StatefulElement的构造函数（删除一些assert）

Element:
```dart
Element(Widget widget)
    : assert(widget != null),
      _widget = widget;
```

ComponentElement：
```dart
ComponentElement(Widget widget) : super(widget);
```


StatefulElement:
```dart
StatefulElement(StatefulWidget widget)
      : _state = widget.createState(),
        super(widget) {
    state._element = this;
    state._widget = widget;
  }
```
问题1解答：
Widget在布局时，虽然形成依赖关系，但是并不是树形关系，例如：Container的child参数，只能表示Container持有了child，并不一定表示Container会绘制child，虽然他实际上绘制了。只有build方法返回的那个Widget才是他的真实child。在Element的inflateWidget时将Element形成树关系，但是Widget并不形成树关系。

问题2解答：
从StatefulElement源码可知，state被绑定到与StatefulWidget对应的StatefulElement上。Widget可以生成多个Element，但是state与StatefulElement却是一对一的关系。

## initState
StatefulElement在mount之后，触发_firstBuild:
```dart
  @override
  void _firstBuild() {
    assert(state._debugLifecycleState == _StateLifecycle.created);
    try {
      _debugSetAllowIgnoredCallsToMarkNeedsBuild(true);
      final Object? debugCheckForReturnedFuture = state.initState() as dynamic;
    } finally {
      _debugSetAllowIgnoredCallsToMarkNeedsBuild(false);
    }
    state.didChangeDependencies();
    super._firstBuild();
  }

```

## didChangeDependencies
从上述代码可以看出，didChangeDependencies紧随initState后会调用一次，但是肯定还有其他触发点。

接着进入_firstBuild流程。触发didChangeDependencies。
```dart
  @override
  void performRebuild() {
    if (_didChangeDependencies) {
      state.didChangeDependencies();
      _didChangeDependencies = false;
    }
    super.performRebuild();
  }
```
这里_didChangeDependencies的默认值为false，并不会触发调用。

didChangeDependencies中可以执行涉及InheritedWidget的初始化，因为在InheritedWidget发生变化并发送通知后，该Element.didChangeDependencies会被调用，

Element：
```dart
 void didChangeDependencies() {
    markNeedsBuild();
  }
```

///StatelessElement
```dart
 @override
  void didChangeDependencies() {
    super.didChangeDependencies();
    _didChangeDependencies = true;
  }
```
Element被标记dirty，下一帧rebuild=》performRebuild，这样就会执行到didChangeDependencies。

## build
build在Element的生命周期中可以多次被调用，一共有4种情况：
1. 首次inflate。
2. 父Element重新build可能会触发。
3. 依赖的InheritedWidget发生变化会触发。
4. 自身被setState标记需要从新绘制。

```
  @override
  Widget build() => state.build(this);
```

## didUpdateWidget
在父Element=》updateChild时会触发当前Element=》update

StatefulElement：
```dart
  @override
  void update(StatefulWidget newWidget) {
    super.update(newWidget);
    final StatefulWidget oldWidget = state._widget!;
    // We mark ourselves as dirty before calling didUpdateWidget to
    // let authors call setState from within didUpdateWidget without triggering
    // asserts.
    _dirty = true;
    state._widget = widget as StatefulWidget;
    try {
      _debugSetAllowIgnoredCallsToMarkNeedsBuild(true);
      final Object? debugCheckForReturnedFuture = state.didUpdateWidget(oldWidget) as dynamic;
    } finally {
      _debugSetAllowIgnoredCallsToMarkNeedsBuild(false);
    }
    rebuild();
  }
```

此时会触发State的didUpdateWidget和build方法。

那么事件会沿树向下传递吗？答案是会，但是可以阻断：

Element:
```dart
  @protected
  @pragma('vm:prefer-inline')
  Element? updateChild(Element? child, Widget? newWidget, Object? newSlot) {
    if (newWidget == null) {
      if (child != null)
        deactivateChild(child);
      return null;
    }
    final Element newChild;
    if (child != null) {
      bool hasSameSuperclass = true;
        ///1. 不更新Element
      if (hasSameSuperclass && child.widget == newWidget) {
        if (child.slot != newSlot)
          updateSlotForChild(child, newSlot);
        newChild = child;
      } else if (hasSameSuperclass && Widget.canUpdate(child.widget, newWidget)) {
        if (child.slot != newSlot)
          updateSlotForChild(child, newSlot);
        ///2. 更新Element  
        child.update(newWidget);
        assert(child.widget == newWidget);
        assert(() {
          child.owner!._debugElementWasRebuilt(child);
          return true;
        }());
        newChild = child;
      } else {
        deactivateChild(child);
        assert(child._parent == null);
        ///3. 重新挂载Element
        newChild = inflateWidget(newWidget, newSlot);
      }
    } else {
        ///4. 重新挂载Element
      newChild = inflateWidget(newWidget, newSlot);
    }
    return newChild;
  }

```
注释1于2的主要区别child.widget是否等于newWidget，当我们声明Widget为const时，就会进入流程1，对于效率有提升，如果是非const，那么会一直沿着树节点向下传递到叶子节点。

## deactivate
当Element需要被卸载时调用，还是看updateChild代码，当child==null时卸载Element。卸载会沿着当前节点向下传递。但是什么情况会child==null呢？

## dispose
在Element=》unmount时调用，unmount在WidgetBinding=》 drawFrame最后调用buildOwner!.finalizeTree()，一次性unmount所有deactivate的Element。
