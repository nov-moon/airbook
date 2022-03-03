# Navigator渲染第一个页面
一个普通Flutter的页面层级。
```
MaterialApp
    --WidgetsApp
        --Navigator
            --Overlay
```

NavigatorState
```dart
  ///暂时没发现和第一帧渲染有关的东西
  void initState(){}

  ///暂时没发现和第一帧渲染有关的东西
  @override
  void didChangeDependencies() {
    super.didChangeDependencies();
    _updateHeroController(HeroControllerScope.of(context));
    for (final _RouteEntry entry in _history)
      entry.route.changedExternalState();
  }
  
  @override
  Widget build(BuildContext context) {
    assert(!_debugLocked);
    assert(_history.isNotEmpty);
    // Hides the HeroControllerScope for the widget subtree so that the other
    // nested navigator underneath will not pick up the hero controller above
    // this level.
    return HeroControllerScope.none(
      child: Listener(
        onPointerDown: _handlePointerDown,
        onPointerUp: _handlePointerUpOrCancel,
        onPointerCancel: _handlePointerUpOrCancel,
        child: AbsorbPointer(
          absorbing: false, // it's mutated directly by _cancelActivePointers above
          child: FocusScope(
            node: focusScopeNode,
            autofocus: true,
            child: UnmanagedRestorationScope(
              bucket: bucket,
              child: Overlay(
                key: _overlayKey,
                ///通过_allRouteOverlayEntries获取到需要绘制的布局
                initialEntries: overlay == null ?  _allRouteOverlayEntries.toList(growable: false) : const <OverlayEntry>[],
              ),
            ),
          ),
        ),
      ),
    );
  }
  
  Iterable<OverlayEntry> get _allRouteOverlayEntries sync* {
    for (final _RouteEntry entry in _history)
      yield* entry.route.overlayEntries;
  }

  ///确定初始化路由  
  @override
  void restoreState(RestorationBucket? oldBucket, bool initialRestore) {

///这两句是存储什么东西，不用管
    registerForRestoration(_rawNextPagelessRestorationScopeId, 'id');
    registerForRestoration(_serializableHistory, 'history');

    // Delete everything in the old history and clear the overlay.
    while (_history.isNotEmpty) {
      _history.removeLast().dispose();
    }
    assert(_history.isEmpty);
    _overlayKey = GlobalKey<OverlayState>();

    // Populate the new history from restoration data.对于第一次启动的app，这里也不会添加任何东西
    _history.addAll(_serializableHistory.restoreEntriesForPage(null, this));
    ///使用pages api时widget.pages才不空
    for (final Page<dynamic> page in widget.pages) {
      final _RouteEntry entry = _RouteEntry(
        page.createRoute(context),
        initialState: _RouteLifecycle.add,
      );
      _history.add(entry);
      _history.addAll(_serializableHistory.restoreEntriesForPage(entry, this));
    }

    // If there was nothing to restore, we need to process the initial route.
    if (!_serializableHistory.hasData) {
    ///widget.initialRoute就是MaterialApp配置的initialRoute
      String? initialRoute = widget.initialRoute;
      if (widget.pages.isEmpty) {
        initialRoute = initialRoute ?? Navigator.defaultRouteName;
      }
      if (initialRoute != null) {
        _history.addAll(
      ///widget.onGenerateInitialRoutes 就是MaterialApp配置的onGenerateInitialRoutes，如果使用GetX框架的话
      ///List<Route<dynamic>> initialRoutesGenerate(String name) =>[PageRedirect(RouteSettings(name: name), unknownRoute).page()];
      ///到这里_history就有了第一个_RouteEntry了。
          widget.onGenerateInitialRoutes(
            this,
            widget.initialRoute ?? Navigator.defaultRouteName,
          ).map((Route<dynamic> route) => _RouteEntry(
              route,
              initialState: _RouteLifecycle.add,
              restorationInformation: route.settings.name != null
                ? _RestorationInformation.named(
                  name: route.settings.name!,
                  arguments: null,
                  restorationScopeId: _nextPagelessRestorationScopeId,
                )
                : null,
            ),
          ),
        );
      }
    }
  }
```

RestorationMixin
```dart
  @override
  void didChangeDependencies() {
    super.didChangeDependencies();

    final RestorationBucket? oldBucket = _bucket;
    final bool needsRestore = restorePending;
    _currentParent = RestorationScope.of(context);

    final bool didReplaceBucket = _updateBucketIfNecessary(parent: _currentParent, restorePending: needsRestore);

    if (needsRestore) {
      _doRestore(oldBucket);
    }
    if (didReplaceBucket) {
      assert(oldBucket != _bucket);
      oldBucket?.dispose();
    }
  }
  
    void _doRestore(RestorationBucket? oldBucket) {
    restoreState(oldBucket, _firstRestorePending);
    _firstRestorePending = false;
  }
```

在build方法中使用_allRouteOverlayEntries来填充Overlay，_allRouteOverlayEntries是Getter方法，具体逻辑是从_history获取enrties.

1. initState: 做的事情与第一个页面无关。
2. didChangeDependencies：这个比较关键,看调用链：NavigatorState.didChangeDependencies() ->RestorationMixin.didChangeDependencies() -> RestorationMixin._doRestore() -> NavigatorState.restoreState(),最终第一个路由的确定就是在这里。
3. build：_allRouteOverlayEntries -> 遍历_history列表，获取到overlayEntries。
4. Overlay会根据OverlayEntry的透明度属性和前后顺序绘制对应的widget。
