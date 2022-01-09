写文章

登录

# 深入 Dart mixin 机制



在 Dart 语言中，我们经常可以看到对 `mixin`关键字的使用，根据字面理解，就是混合的意思。那么，`mixin`如何使用，它的使用场景是什么呢。

## 从一个实例说起

我们假设一个需求，我们需要用多个对象表示一些 **动物**， 诸如 狗、鸟、鱼、青蛙。其中

1. 狗会跑
2. 鸟会飞
3. 鱼会游泳
4. 青蛙是两栖动物，会跑，并且会游泳

基于如下一些考虑

- 动物特性可能会继续增多，并且一个动物可能具备多种技能
- 动物种类很多，但是可以归大类。例如 鸟禽、哺乳类

我们使用如下设计

- 动物继承自 **Animal**抽象类
- 跑、飞、游 抽象为接口

代码如下：

```text
abstract class Animal {
}

class Run {
    run() {
        print('run');
    }
}

class Fly {
    fly() {
        print('fly');
    }
}

class Swim {
    swim(){
        print('swim');
    }
}

class Bird extends Animal implements Fly {
    @override
    fly() {
        super.fly();
    }
}

class Dog extends Animal implements Run {
    @override
    run() {
        super.run();
    }
}

class Fish extends Animal implements Swim {
    @override
    swim() {
        super.swim();
    }
}

class Frog extends Animal implements Run,Swim {
    @override
    run() {
        super.run();
    }

    @override
    swim() {
        super.swim();
    }
}
```

这个时候，我们会发现编辑器报了个错

原来这个方法 Dart 会一直认为 `super`调用是在调用一个 abstract 的函数，所以我们这时候需要把这里面集成的函数实现一一实现。

这时候问题来了，Frog 和 Fish 都实现了 Swim 接口，这时候 swim 函数的内容我们需要重复的写 2 遍！

回想一下我们当初在 Android 中写 Java 或者 Kotlin 的时候，其实也有类似问题，同一个 interface 内的 method， 我们可能需要重写 n 次，非常明显的代码冗余。

Java8 和 Kotlin 选择使用接口的 default 实现来解决这个问题：

```text
interface IXX {
    default void xmethod() {
        /// do sth...
    }
}
```

而 Dart， 选择使用 `mixin`

修改上面的代码：

```text
abstract class Animal {
}

mixin Run {
    run() {
        print('run');
    }
}

mixin Fly {
    fly() {
        print('fly');
    }
}

mixin Swim {
    swim(){
        print('swim');
    }
}

class Bird extends Animal with Flym {}
class Dog extends Animal with Run {}
class Fish extends Animal with Swim {}
class Frog extends Animal with Run,Swim {}
```

我们运行如下代码

```text
Bird bird = Bird();
bird.fly();

Frog frog = Frog();
frog.run();
frog.swim();
```

输出如下：

```text
fly
run
swim
```

这里我们可以意识到，`mixin`被混入到了具体的类中，实际也起到了实现具体特性的作用。但是相比实现接口来说，更加的便捷一点。

这里类的继承关系我们可以梳理成下图

## 当函数一样的时候

上述的例子结束了 `mixin`的基本用法。我们可以看到每个类都可以通过 `with`关键字，把 `mixin`中定义的特性 “混入” 到自己这里来。但是这时候如果每个 `mixin`的函数名是一样的，会发生什么呢？我们不妨重新写一个简单的例子。

```text
class S {
  fun()=>print('A');
}
mixin MA {
  fun()=>print('MA');
}
mixin MB {
  fun()=>print('MB');
}
class A extends S with MA,MB {}
class B extends S with MB,MA {}
```

运行如下代码

```text
main() {
A a = A();
a.fun();
B b = B();
b.fun();
}
```

我们得到下面这个输出

```text
MB
MA
```

这个时候我们会发现，最后混入的 `mixin`的函数，被调用了。这说明最后一个混入的 `mixins`会覆盖前面一个 `mixins`的特性。为了验证这个工作流程，我们稍微修改一下这个例子，给 `mixins`的函数加上 super 调用。

```text
mixin MA on S {
  fun() {
    super.fun();
    print('MA');
  }
}
mixin MB on S {
  fun() {
    super.fun();
    print('MB');
  }
}
```

继续执行上面的程序，输出结果如下

```text
A
MA
MB
A
MB
MA
```

第一个 `A#fun`为例子。我们发现实际的调用顺序为 MB -> MA -> A，这里我们可以看出来 `mixin`的工作方式，是具有线性化的。

### mixin的线性化

上面的示例，我们可以画一个图来表示 `mixin`是如何线性化的

Dart 中的 `mixin`通过创建一个类来实现，该类将 `mixin`的实现层叠在一个超类之上以创建一个新类 ，它不是“在超类中”，而是在超类的“顶部”。

我们可以得到以下几个结论：

1. `mixin`可以实现类似多重继承的功能，但是实际上和多重继承又不一样。多重继承中相同的函数执行并不会存在 ”父子“ 关系
2. `mixin`可以抽象和重用一系列特性
3. `mixin`实际上实现了一条继承链

最终我们可以得出一个很重要的结论

**声明 mixin 的顺序代表了继承链的继承顺序，声明在后面的 mixin，一般会最先执行**

这里再提出一个假设，如果 MA 和 MB 都有一个函数叫 `log`， 如果在先声明的 `mixin`中执行 `log`函数，会发生声明事情呢？

代码如下

```text
mixin MA on S {
  fun() {
    super.fun();
    log();
    print('MA');
  }

  log() {
    print('log MA');
  }
}
mixin MB on S {
  fun() {
    super.fun();
    print('MB');
  }

  log() {
    print('log MB');
  }
}

class A extends S with MA,MB {}
A a = A();
a.fun();
```

这里按照习惯性的思维，我们可能会得到

```text
A
log MA
MA
MB
```

的结果。实际上，我们的输出是

```text
A
log MB
MA
MB
```

仔细思考一下，按照上面的工作原理，在 `mixin`的继承链建立的时候，最后声明的 `mixin`会把后声明的 `minxin`的函数覆盖掉。这时候即使我们从代码逻辑中认为在 MA 中调用了 `log`函数，实际上这时候 A 类中的 `log`函数已经被 MB 给覆盖了。所以最终，`log`函数调用的是 MB 中的 `log`函数逻辑。

## 类型

根据 `mixin`的工作原理，我们完全可以大胆猜想，最终的子类类型和这个继承链上所有父类和混入的 `mixin`的类型都可以匹配上。我们来验证一下这个猜想：

```text
A a = A();
print(a is A);
print(a is S);
print(a is MA);
print(a is MB);
```

输出结果

```text
true
true
true
true
```

推论完全正确。

## mixin 的使用场景

我们应该在什么时候使用 `mixin`呢？很简单，在我们编写 Java 的时候，感觉需要实现多个 `interface`的时候。

那么，这个和多重继承相比，在某些场景有什么好处吗？答案是有。

在 Flutter 中，framework 的执行依赖多个 `Binding`，我们查看最外层 `WidgetsFlutterBinding`的定义:

```text
class WidgetsFlutterBinding extends BindingBase with GestureBinding, ServicesBinding, SchedulerBinding, PaintingBinding, SemanticsBinding, RendererBinding, WidgetsBinding {}
```

在 `WidgetsBinding`和 `RendererBinding`中，都有一个叫做 `drawFrame`的函数。在 `WidgetsBinding`的 `drawFrame`中，也有 `super.drawFrame()`的调用。

这里 `mixin`的优点就体现了出来，我们可以看到这个逻辑有如下2点

1. 保证了 widget 等的 `drawFrame`先于 render 层的调用，保证了 Flutter 在布局和渲染处理中 widgets -> render 的处理顺序
2. 保证顺序的同时，Widgets 和 Render 仍然属于 2 个不同的对象定义，职责分割的非常的清晰。

具体的细节，感兴趣的同学可以阅读 Flutter 的 flutter package 的源码。

## 小结

这篇文，我对 Dart 的 `mixin`的使用、工作机制、使用场景做了一个大致的总结。`mixin`是一个强大的概念，我们可以跨越类的层次结构重用代码。

文中一些优势和工作机制是我的个人理解。在初次接触 Dart 的这个机制的时候，也需要很多的思维转变。如果文中我有理解的不对的地方，或者您有不同的理解。欢迎评论讨论交流。

发布于 2019-06-02 21:50

赞同 10

5 条评论

分享

喜欢收藏申请转载



- 