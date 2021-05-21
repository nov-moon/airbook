# GetX(三) GetBuilder
往期列表：
* [Obx中的魔法](https://juejin.cn/post/6934628323712892941/)
* [GetX中的GetxController和GetX控件](https://juejin.cn/post/6935418081372012581/)

在上一节中我们一块学习了GetX中的GetxController的源码和作用，以及GetX的详细解读。

在本文中，我们继续学习官方demo中的另一个批量更新控件：GetBuilder，代码如下：
```dart
class GetBuilder<T extends GetxController> extends StatefulWidget {
    final GetControllerBuilder<T> builder;
    final bool global;
    final Object id;
    final String tag;
    final bool autoRemove;
    final bool assignId;
    final Object Function(T value) filter;
    final void Function(State state) initState, dispose, didChangeDependencies;
    final void Function(GetBuilder oldWidget, State state) didUpdateWidget;
    final T init;

const GetBuilder({
    Key key,
    this.init,
    this.global = true,
    @required this.builder,
    this.autoRemove = true,
    this.assignId = false,
    this.initState,
    this.filter,
    this.tag,
    this.dispose,
    this.id,
    this.didChangeDependencies,
    this.didUpdateWidget,
})  : assert(builder != null),
        super(key: key);

    @override
    _GetBuilderState<T> createState() => _GetBuilderState<T>();
}
```

从上述源码可以看出，GetBuilder的定义和GetX的定义十分相近，每个参数所提供的功能也基本相同。他们的不同之处在于：
* 定义规定的泛型不同
    * `class GetBuilder<T extends GetxController>` 我们知道GetX规定的入参泛型为 **DisposableInterface** 类型，所以GetX的入参只支持生命周期回调，而不支持controller手动更新。我们在分析**GetxController**的源码时介绍过，他继承自**DisposableInterface**，并且又通过mixin的方式融入了批量更新的功能。所以我们推测**GetBuilder支持controller中手动更新**。
* filter参数
* id参数

下面我们具体看一下State的实现，寻找三处不同的具体差别。

## 1. _GetBuilderState源码分析
我们直接进入正题，查看一下_GetBuilderState的源码：
```dart
class _GetBuilderState<T extends GetxController> extends State<GetBuilder<T>> with GetStateUpdaterMixin {
  T controller;
  bool isCreator = false;
  VoidCallback remove;
  Object _filter;

  @override
  void initState() {
    super.initState();
    // 初始化controller，与GetX中的一致，省略...
    
    // 1
    if (widget.filter != null) {
      _filter = widget.filter(controller);
    }

    _subscribeToController();
  }

  // 2
  void _subscribeToController() {
    remove?.call();
    remove = (widget.id == null)
        ? controller?.addListener(
            _filter != null ? _filterUpdate : getUpdate,
          )
        : controller?.addListenerId(
            widget.id,
            _filter != null ? _filterUpdate : getUpdate,
          );
  }

  // 3
  void _filterUpdate() {
    var newFilter = widget.filter(controller);
    if (newFilter != _filter) {
      _filter = newFilter;
      getUpdate();
    }
  }

  @override
  Widget build(BuildContext context) {
    return widget.builder(controller);
  }
}
mixin GetStateUpdaterMixin<T extends StatefulWidget> on State<T> {
  void getUpdate() {
    if (mounted) setState(() {});
  }
}
```
从源码我们看到，GetBuilder中的区别从上到下依次为：
* **mixin GetStateUpdaterMixin** 从GetStateUpdaterMixin中的代码我们知道，其实并没有做太多事情，只是增加了state状态判断以及简化了state的调用。
* **listener的注册** 当init执行完成后，会根据当前widget.id是否为null，在controller中注册更新监听。而监听的回调会根据filter的状态，决定是否更新UI。例如，我们的controller是多个GetBuilder共享，当controller更新时，可使用refreshGroup()的方式批量更新对应id的GetBuilder，使用refresh()的方式更新没有指定id的GetBuilder。
* **filter** 根据注释1我们知道，当init的时候会第一次调用filter。
    * 如果返回值不为空，则后续的更新回调，将使用**_filterUpdate()**的方式进行。而此方法中，又会回调filter，让外部决定是否更新。如果新返回的值与原始值相同，则不更新，否则更新。
    * 如果返回值为空，则直接使用mixin进来的getUpdate方法进行更新
* **build方法** 从源码可以看到，这里并没有像GetX与Obx中那样使用proxy交换的方式远程注册监听，而是直接调用的**widget.builder(controller)**。由此代码以及listener代码我们可知，GetBuilder不支持自动更新UI，需要配合controller中的refresh、refreshGroup进行UI更新。

## 2. 总结
通过上面的源码阅读与分析我们知道，GetBuilder的设计是用来与GetxController进行配合使用。在需要批量更新和区别更新(refreshGroup)时，会比GetX和Obx更加灵活。

GetBuilder与GetX等不同，他不支持自动更新。需要我们在Controller中有数据变化和更新需求时，手动的进行更新。

GetBuilder提供了filter参数，通过此参数，我们可以更加细粒度的控制每次更新触发时，是否真正的执行更新，或者过滤掉更新请求。