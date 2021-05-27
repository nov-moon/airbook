# 组件状态共享
在Flutter的框架中，使用InheritedWidget来实现Widget之间的信息共享。InheritedWidget被描述为在树中高效向下传递信息的Widget基类。使用InheritedWidget我们能够实现：
1. 为子组件提供数据共享。
2. 为子组件提供响应式更新。

## 简单实例
代码片段
```
@override
  Widget buildPage(BuildContext context) {
    Color a = Theme.of(context).accentColor;
    ....
    }
```
Theme使用_InheritedTheme来完成它的状态共享，_InheritedTheme是InheritedWidget的实现。

## 如何实现数据共享

### 如何在Element树中存储需要共享的数据
Flutter框架在InheritedElement挂载时将数据关联起来，以首次mount为例：

![](./第一次如何保存数据.png)

关键方法：_updateInheritance有两个实现，一个在Element，一个在InheritedElement。

```
//Element
  void _updateInheritance() {
    assert(_lifecycleState == _ElementLifecycle.active);
    _inheritedWidgets = _parent?._inheritedWidgets;
  }

//InheritedElement
  @override
  void _updateInheritance() {
    assert(_lifecycleState == _ElementLifecycle.active);
    final Map<Type, InheritedElement>? incomingWidgets = _parent?._inheritedWidgets;
    if (incomingWidgets != null)
      _inheritedWidgets = HashMap<Type, InheritedElement>.from(incomingWidgets);
    else
      _inheritedWidgets = HashMap<Type, InheritedElement>();
    _inheritedWidgets![widget.runtimeType] = this;
  }  

```
整个流程组合起来实现了一件事：每个Element会维护一张_inheritedWidgets表，这张表包括了从自身开始到Element树Root这条路径上所有InheritedElement的{widget.runtimeType,InheritedElement}映射。

### 子Element如何查询到相应数据

BuildContext提供了两种查询接口：
1. `  T? dependOnInheritedWidgetOfExactType<T extends InheritedWidget>({Object? aspect})`：这种查询方式会让自己在InheritedElement更新时，更新自己的展示。将Element自己添加到了_dependencies。
2. `InheritedElement? getElementForInheritedWidgetOfExactType<T extends InheritedWidget>()`：单纯的查询InheritedElement。

这两种的实现都在Element:

```
  @override
  InheritedWidget dependOnInheritedElement(InheritedElement ancestor, { Object? aspect }) {
    assert(ancestor != null);
    _dependencies ??= HashSet<InheritedElement>();
    _dependencies!.add(ancestor);
    //添加数据依赖，以便InheritedWidget数据发生改变后，更新页面。
    ancestor.updateDependencies(this, aspect);
    return ancestor.widget;
  }

  @override
  T? dependOnInheritedWidgetOfExactType<T extends InheritedWidget>({Object? aspect}) {
    assert(_debugCheckStateIsActiveForAncestorLookup());
    final InheritedElement? ancestor = _inheritedWidgets == null ? null : _inheritedWidgets![T];
    if (ancestor != null) {
      assert(ancestor is InheritedElement);
      return dependOnInheritedElement(ancestor, aspect: aspect) as T;
    }
    _hadUnsatisfiedDependencies = true;
    return null;
  }

  @override
  InheritedElement? getElementForInheritedWidgetOfExactType<T extends InheritedWidget>() {
    assert(_debugCheckStateIsActiveForAncestorLookup());
    final InheritedElement? ancestor = _inheritedWidgets == null ? null : _inheritedWidgets![T];
    return ancestor;
  }
```


```
abstract class InheritedWidget extends ProxyWidget {
  const InheritedWidget({ Key? key, required Widget child })
    : super(key: key, child: child);

  @override
  InheritedElement createElement() => InheritedElement(this);

  // 当此方法返回false是，不会更新child。
  @protected
  bool updateShouldNotify(covariant InheritedWidget oldWidget);
}
```

### 数据改变如何通知更新
我们知道，所有的Widget都是`@immutable`。那么改变InheritedWidget的一种方式是父Widget标dirty，用新的数据从新生成InheritedWidget。

![](./InheritedWidget数据变更.png)

最后一个流程：InheritedElement.notifyClients()源码如下，又回到_dependencies找到元素，并标记dirty。

```
  //InheritedWidget
  @override
  void notifyClients(InheritedWidget oldWidget) {
    assert(_debugCheckOwnerBuildTargetExists('notifyClients'));
    for (final Element dependent in _dependents.keys) {
      assert(() {
        // check that it really is our descendant
        Element? ancestor = dependent._parent;
        while (ancestor != this && ancestor != null)
          ancestor = ancestor._parent;
        return ancestor == this;
      }());
      // check that it really depends on us
      assert(dependent._dependencies!.contains(this));
      notifyDependent(oldWidget, dependent);
    }
  }

  //Element
  void rebuild() {
    assert(_lifecycleState != _ElementLifecycle.initial);
    //下面这个判断当_dirty为false时，不继续流程。
    if (_lifecycleState != _ElementLifecycle.active || !_dirty)
      return;
    .......
  }
```

## 应用实例

### Theme
Theme是一个StatelessWidget类型的组件，存储Theme数据的是一个final类型的ThemeData变量。而且ThemeData也被标记为@immutable。那么就很明显了，不存在单独改变某个ThemeData的属性来修改Theme的情况，只能重新构建Theme对象来改变属性。

```
//将ThemeData与字体相关的本地化配置合并生成新的ThemeData返回给用户。
static ThemeData of(BuildContext context) {
    final _InheritedTheme? inheritedTheme = context.dependOnInheritedWidgetOfExactType<_InheritedTheme>();
    final MaterialLocalizations? localizations = Localizations.of<MaterialLocalizations>(context, MaterialLocalizations);
    final ScriptCategory category = localizations?.scriptCategory ?? ScriptCategory.englishLike;
    final ThemeData theme = inheritedTheme?.theme.data ?? _kFallbackTheme;
    return ThemeData.localize(theme, theme.typography.geometryThemeFor(category));
  }

  @override
  Widget build(BuildContext context) {
    return _InheritedTheme(
      theme: this,
      child: CupertinoTheme(
        data: MaterialBasedCupertinoThemeData(
          materialTheme: data,
        ),
        child: IconTheme(
          data: data.iconTheme,
          child: child,
        ),
      ),
    );
  }

```

#### Theme默认值

在ThemeData的构造方法中有很多判空，并给定默认值：`primarySwatch ??= Colors.blue;`

### Theme的使用
从代码结构来看，ThemeData并没有传递性，如果你想用一个新的局部Theme来改变全局Theme的某一个属性是做不到的。新Theme的child不能能查询到全局Theme的ThemeData。
1. 系统默认使用：在很多系统组件中，某些属性如果没有赋值，那么会自动查询Theme赋值，例如TextFormField：`final InputDecoration effectiveDecoration = (decoration ?? const InputDecoration())
             .applyDefaults(Theme.of(field.context).inputDecorationTheme);`
2. 主动使用：Theme.of(context)获取ThemeData，直接使用属性即可。     

