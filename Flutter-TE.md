> **看完本篇，你不仅会了解到 TextField 的实现和构成，还可以学到很多之前不常用的“奇怪”知识**。



在 Flutter 里 `TextField` 是一个比较复杂的控件，而在整个 `TextField` 里嵌套了许多不同实现的控件，它们组成了我们常用的输入框效果，**如下图所示是关于 `TextField` 的主要构成部分**，也是本篇主要讲解的内容。


![](http://img.cdn.guoshuyu.cn/20211223_Flutter-TE/image1)


## FocusTrapArea 

`FocusTrapArea` 大家可能会比较陌生，这个是最近的版本里才出现的控件，`FocusTrapArea` 本身并没有特别，它仅仅是在 `RenderObject` tree 里塞进去了一个 `FocusNode`。

它的出现主要是为了 Web/Desktop 平台，通过增加了 `FocusTrapArea`  之后，在  Web/Desktop 平台执行 `TextEditingController.clear` 的时候，`TextField` 还能继续保持之前获得的焦点。


> 具体可见 Flutter 的 issues ： [#86154](https://github.com/flutter/flutter/issues/86154) 、[#86041](https://github.com/flutter/flutter/pull/86041)

正常效果                                                                                                                                                                                                       |   非正常效果                                                                                                                                                                                       |
| --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [![stable](http://img.cdn.guoshuyu.cn/20211223_Flutter-TE/image2)](https://user-images.githubusercontent.com/140617/125034739-3c363f80-e091-11eb-8907-9fb256816c2d.gif) | [![master](http://img.cdn.guoshuyu.cn/20211223_Flutter-TE/image3)](https://user-images.githubusercontent.com/140617/125034754-40faf380-e091-11eb-9cfc-9a87777df863.gif) |

## MouseRegion

顾名思义是用于处理鼠标相关事件，主要用于响应鼠标独占的 Pointer事件，比如：鼠标进入/离开控件区域、光标显示效果等等。

## IgnorePointer

它在 `TextField` 里主要用于处理**当前输入框是否可用的的状态**，比如当 `widget.enabled` 或者  `widget.decoration?.enabled` 为 `false` 时，`IgnorePointer` 就会屏蔽整个区域内的手势事件，从而让 `TextField` 会无法点击输入。


## TextSelectionGestureDetectorBuilder


关于 `TextSelectionGestureDetectorBuilder` 大家应该比较少接触，而在 `TextField` 里使用的是它的子类 `_TextFieldSelectionGestureDetectorBuilder`：

> **它主要是处理 `TextField` 内针对 `EditableText` 的点击、滑动、长按等事件，例如单击弹起键盘，长按弹出选择复制/粘贴框等等**。

在 `TextSelectionGestureDetectorBuilder` 的内部主要是通过 `editableTextKey` 这个  `GlobalKey` 去获取到 `EditableTextState `，从而将各种手势事件和 `EditableText` 里的行为关联起来。

> 该控件内部使用的是 `TextSelectionGestureDetector` 。

例如在 `_TextFieldSelectionGestureDetectorBuilder` 中，可以看到 `onSingleTapUp` 的处理流程：


![](http://img.cdn.guoshuyu.cn/20211223_Flutter-TE/image4)

如上代码所示：

- 1、收起已经弹出的 Toolbar （一个 `Overlay`，也就是复制/粘贴之类的弹框）；
- 2、根据不同平台选择响应事件；
- 3、执行弹出键盘操作；
- 4、回调点击事件；

所以可以看到，**这里其实是先执行弹出键盘，然后再回调点击的 callback**，所以如果你需要在点击弹出键盘前，针对 `TextField` 作一些处理，那么 `TextField`  的 `onTap` 其实并不合适，因为它是已经弹出了。

**最后 `_TextFieldSelectionGestureDetectorBuilder` 会调用 `buildGestureDetector` 方法生成一个监听和处理触摸的控件，用于嵌套 child**。


## InputDecorator

关于 `InputDecorator` 的内部参数解析这里就不多说，以前在书里已经有详细介绍过，用过 `TextField` 的大家对于 `InputDecorator` 应该也不会陌生，在 **`TextField` 里 `InputDecorator` 的实现是和 `AnimatedBuilder` 一起组成使用**。

因为在 `TextField`  里 `FocusNode` 和 `TextEditingController` 都是 `ChangeNotifier`（`Listenable`) ，所以它们可以被用于 `AnimatedBuilder` 的  `animation`。

![](http://img.cdn.guoshuyu.cn/20211223_Flutter-TE/image5)


**也就是当 `FocusNode` 和 `TextEditingController` 这两者发生改变的时候，会让 `InputDecorator` 重新 `rebuild` 从而改变渲染效果**，例如：输入框输入内容时、焦点发生改变时修改输入框的背景颜色。


> 注意别搞混了 `InputDecorator`  和 `InputDecoration`，`InputDecoration` 是用来配置 `InputDecorator`。


![](http://img.cdn.guoshuyu.cn/20211223_Flutter-TE/image6)


所以可以看到 `InputDecorator` 有很丰富的参数和配置，开发者可以通过 `InputDecoration` 来配置很丰富的输入框 UI 效果，**但是如果刚好出现某些位置，或者某些缝隙不满足产品诡异的需求时，那恭喜你，你开启了 Flutter 高级开发的修炼之路**。

为什么呢？

简单来说 `InputDecorator` 的实现是在内部是一个自定义的 `RenderBox`，其中和 layout 相关部分就有 600 多行的代码，也就是根据 `InputDecoration` 的 `icon`、`prefixIcon`、`suffix` 等参数，进行定位布局，计算位置方向，根据基线调整位置等等。

> 另外` InputDecorator` 里的动画效果主要是通过内部的 `AnimatedOpacity` 等完成。

所以对于 `InputDecorator` 来说，如果你对于某些位置或者边界效果不满意，要么你就重构一个自己的实现，要么可能就要选择“委曲求全”。


## RepaintBoundary

为什么 `TextField` 内部会有一个 `RepaintBoundary` ？ 首先 `RepaintBoundary` 是干嘛的？

之前在 [《Flutter 画面渲染的全面解析》](https://juejin.cn/post/6844904104452440072) 详细介绍过这部分的知识，这简单不严谨地说就是： **`RepaintBoundary` 主要是用于形成一个 `Layer`，得到一个独立的绘制区域**。

常见的就是 `Navigator` 的页面跳转，内部基础实现都有一个 `RepaintBoundary` 来保证每个区域都是独立的绘制区域。

> 另外说到 `Navigator`就不得不说每个页面也都有自己的 `FocusScope`， 也就是我们常用的  `FocusScope.of(context)` 等用于键盘和焦点处理。


在  `TextField` 内部有一个 `RepaintBoundary` ，是因为 `TextField` 本身是一个需要频繁更新的控件，而 `TextField` 里的内容变化一般很少需要触发父布局的重绘，**所以 `RepaintBoundary` 的存在让 `TextField` 可以实现性能更好的局部绘制**。



## UnmanagedRestorationScope

`UnmanagedRestorationScope` 大家可能比较少用到，它本身是一个 `InheritedWidget` ，主要是往下共享一个 `RestorationBucket` ，而 **`RestorationBucket` 主要是和实现状态的保存/恢复有关系**。

例如应用因为低内存在后台被回收时，可以通过它在重新回到 App 时恢复指定的数据，举个例子：


```dart
import 'package:flutter/material.dart';

void main() => runApp(MyApp());

class MyApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      // Give your RootRestorationScope an id, defaults to null.
      restorationScopeId: 'root', 
      home: HomePage(),
    );
  }
}

class HomePage extends StatefulWidget {
  @override
  _HomePageState createState() => _HomePageState();
}

// Our state should be mixed-in with RestorationMixin
class _HomePageState extends State<HomePage> with RestorationMixin {

  // For each state, we need to use a restorable property
  final RestorableInt _index = RestorableInt(0);

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: Center(child: Text('Index is ${_index.value}')),
      bottomNavigationBar: BottomNavigationBar(
        currentIndex: _index.value,
        onTap: (i) => setState(() => _index.value = i),
        items: <BottomNavigationBarItem>[
          BottomNavigationBarItem(
              icon: Icon(Icons.home),
              label: 'Home'
          ),
          BottomNavigationBarItem(
              icon: Icon(Icons.notifications),
              label: 'Notifications'
          ),
          BottomNavigationBarItem(
              icon: Icon(Icons.settings),
              label: 'Settings'
          ),
        ],
      ),
    );
  }

  @override
  // The restoration bucket id for this page,
  // let's give it the name of our page!
  String get restorationId => 'home_page';

  @override
  void restoreState(RestorationBucket? oldBucket, bool initialRestore) {
    // Register our property to be saved every time it changes,
    // and to be restored every time our app is killed by the OS!
    registerForRestoration(_index, 'nav_bar_index');
  }
}
```



如上代码所示：

- 首先给 `MaterialApp`  配置 `restorationScopeId`（必须配置才算开启该功能）。
- 使用 `RestorableInt` 用于配置和保存 `BottomNavigationBar` 的 `index` ；
- 在 `State` 混入 `RestorationMixin` 并且在 `restoreState` 方法里恢复 `index` 的状态；

其中默认 `MaterialApp` 内部用到了 `RootRestorationScope`， 而`RootRestorationScope` 的内部就是 `UnmanagedRestorationScope`；上述例子运行后通过打开模拟器开发者设置里的 *`Don't keep activities`* 就可以看到效果。

> 以上示例来自 [《Introduction to State Restoration in Flutter》](https://dev.to/pedromassango/what-is-state-restoration-and-how-to-use-it-in-flutter-5blm) 。


回到 `TextField`，在 `_TextFieldState` 里就混入了 `RestorationMixin`，然后使用 `RestorableTextEditingController` 用于用于恢复 `TextEditingController` 。

> 因为输入框的内容默认保存在了 `TextEditingController` 的 `TextEditingValue` 里，所以这里用的是 `RestorableTextEditingController` 。

![](http://img.cdn.guoshuyu.cn/20211223_Flutter-TE/image7)

一般情况下是使用 `MaterialApp` 内部默认自带了一个 `RootRestorationScope` ，所以我们只需要给 `MaterialApp` 设置 `restorationScopeId`，而 **`TextFild` 通过内置  `UnmanagedRestorationScope` 相关的逻辑，最终实现了文本内容的保存与恢复**。



## EditableText

`EditableText` 就不用多说了，`TextField` 的本体，内部主要通过 `Scrollable` 来实现滑动，同样的它也用了对应的 `restorationId` 来实现恢复和缓存。

**首先注意到可以滑动这一点，可以看到对于 `EditableText` 来说，它其实是一个 “ViewPort”，是根据 `ViewportOffset` 来实现滑动效果**。


而对于 `EditableText` 内部，**它使用了 `CompositedTransformTarget` 来实现 Toolbar 和输入框的联动**，也就是输入控件和长按“粘贴/复制”弹出框之间的关联。

**所以这里简单介绍下 `CompositedTransformTarget`，它通常和 `CompositedTransformFollower` 一起被用于控件之间的联动效果**。

![](http://img.cdn.guoshuyu.cn/20211223_Flutter-TE/image8)

如上图所示，常见内置的 `Slider`，在滑动的弹出部分实现，就是通过 `CompositedTransformTarget` 和 `CompositedTransformFollower` 的结合实现，**它可以让一个控件跟随另外一个控件而无需计算位置，它们之间主要是通过 `LayerLink` 链接在一起**。

回到 `TextField`，其实除了 “复制/粘贴” 的 Toolbar ，关于 selection 选中区域的内容，`EditableText` 内部也是通过类似的方式实现，只是这里是直接通过 `LeaderLayer` 而不是通过它的封装 `CompositedTransformTarget` 去实现。

> 对于使用 `CompositedTransformTarget` 有兴趣的可以参考：https://juejin.cn/post/6946416845537116190

当然使用 `CompositedTransformTarget` 还是会有“比较大”的性能开销，不建议大规模频繁使用，因为毕竟它属于一个 `pushLayer` 的操作。

另外 `EditableText`  内部绘制内容的部分，主要就是大家都知道的 `TextPainter` ，这部分就没什么特别，暂时不详细展开。

**所以本篇主要是通过介绍 `TextField` 的组成，以及解释内部各组成部分的作用，让开发者可以更清晰的了解 Flutter 里常用的文本输入框的实现，当遇上问题或者需求时，可以快速定位和解决问题**，例如：

- ”粘贴/复制“ 的 Toolbar 是哪里弹出；
- Toolbar 是如何定位和布局；
- 点击 `TextField` 是如何弹出键盘和处理手势事件；
- `TextField` 如何做到局部绘制；
- ...


最后介绍一个简单的问题，之前有人刚好问我：***如何在 Flutter 上实现类似微信聊天输入框从一行到多行的输入框效果***，如下图代码所示，就是这么简单：

```dart
TextField(
  focusNode: _focusNode,
  maxLines: 7,
  minLines: 1,
  decoration:
      const InputDecoration(border: OutlineInputBorder()),
)

```


![](http://img.cdn.guoshuyu.cn/20211223_Flutter-TE/image9)