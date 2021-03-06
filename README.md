![Supported Platforms](https://img.shields.io/badge/platform-macOS%20%7C%20Linux%20%7C%20Windows-green.svg)
[![Build Status](https://travis-ci.org/downdemo/Design-Patterns-in-Cpp17.svg?branch=master)](https://travis-ci.org/downdemo/Design-Patterns-in-Cpp17)
[![GitHub license](https://img.shields.io/badge/license-MIT-blue.svg)](https://github.com/downdemo/Design-Patterns-in-Cpp17/LICENSE)

## 前言

* 为了练习使用 [std::shared_ptr](https://en.cppreference.com/w/cpp/memory/shared_ptr)、[std::unique_ptr](https://en.cppreference.com/w/cpp/memory/unique_ptr)、[std::weak_ptr](https://en.cppreference.com/w/cpp/memory/weak_ptr)，基于 C++17 实现 23 种 GoF 设计模式
* 使用智能指针并不意味着一定没有内存泄漏，比如循环引用的情况

```cpp
class B;

class A {
 public:
  std::shared_ptr<B> b;
};

class B {
 public:
  std::shared_ptr<A> a;
};

int main()
{
  {
    auto x = std::make_shared<A>();
    x->b = std::make_shared<B>();
    x->b->a = x;
  } // x 的引用计数由 2 减为 1，不会析构，于是 x->b 也不会析构，导致两次内存泄漏
  // 解决方案是将 B::a 改为 std::weak_ptr，这样引用计数不会为2，而是保持 1，此处就会由 1 减为 0，从而正常析构
}
```

* 最初实现观察者模式时，就由于没有正确区分几种智能指针的使用场景，无意识地写出了循环引用的代码

### 检测内存泄漏的方法

* 开发环境使用 [Visual Studio](https://visualstudio.microsoft.com/zh-hans/?rr=https%3A%2F%2Fcn.bing.com%2F)
* 在程序结束点调用 [_CrtDumpMemoryLeaks()](https://docs.microsoft.com/zh-cn/previous-versions/d41t22sb(v=vs.120)?redirectedfrom=MSDN)

```cpp
#include <crtdbg.h>

#ifdef _DEBUG
#define new new(_NORMAL_BLOCK,__FILE__,__LINE__)
#endif

int main()
{
  int* p = new int(42);
  _CrtDumpMemoryLeaks();
}
```

* 调试时将提示第 9 行产生 4 字节的内存泄漏

```cpp
Detected memory leaks!
Dumping objects ->
xxx.cpp(9) : {88} normal block at 0x008C92D0, 4 bytes long.
 Data: <*   > 2A 00 00 00 
Object dump complete.
```

* 这种方法的原理是，在执行此函数时，检查所有未回收的内存，因此存在析构函数还未执行而误报的情况

```cpp
#include <crtdbg.h>
#include <memory>

#ifdef _DEBUG
#define new new(_NORMAL_BLOCK,__FILE__,__LINE__)
#endif

int main()
{
  auto p = std::make_shared<int>(42);
  _CrtDumpMemoryLeaks(); // 此时 std::shared_ptr 还未析构，因此报告内存泄漏
}
```

* 更好的方法是在开始点使用 [_CrtSetDbgFlag()](https://docs.microsoft.com/zh-cn/previous-versions/5at7yxcs(v=vs.120)?redirectedfrom=MSDN)，它在程序退出时调用 [_CrtDumpMemoryLeaks()](https://docs.microsoft.com/zh-cn/previous-versions/d41t22sb(v=vs.120)?redirectedfrom=MSDN)

```cpp
#include <crtdbg.h>

#ifdef _DEBUG
#define new new(_NORMAL_BLOCK,__FILE__,__LINE__)
#endif

int main()
{
  _CrtSetDbgFlag(_CRTDBG_LEAK_CHECK_DF | _CrtSetDbgFlag(_CRTDBG_REPORT_FLAG));
  int* p = new int(42);
}
```

* 该项目中的源码使用上述方式检测均无内存泄漏
* 为了尽可能简单，所有代码均以单个源文件的形式实现
* 源码中未使用平台相关 API，因此在任何支持 C++17 标准的平台均可通过编译

### [std::shared_ptr](https://en.cppreference.com/w/cpp/memory/shared_ptr) 的传参方式选择

* 以下两种传参方式，应该如何选择？

```cpp
void f(std::shared_ptr<A> p); // 传值
void f(const std::shared_ptr<A>& p); // 传引用
```

* 通常情况下，采用传引用方式是最稳妥且没有心智负担的
* 传值方式只在之后一定会对其拷贝时才有使用的可能

```cpp
void f(std::shared_ptr<A> p)
{
  auto q = std::move(p);
  // ...
}
```

* 这种情况常见于构造函数中

```cpp
class X {
 public:
  explicit X(std::shared_ptr<A> _p) : p(std::move(_p)) {}
 private:
  std::shared_ptr<A> p;
};
```

* 为了练习，此项目中均使用智能指针作为参数，实际情况下，原始指针由于方便和低开销的优势，仍有使用的必要，如何选择应当视情况权衡

## 设计模式简介

* 设计模式的概念最初源自建筑行业，建筑师 Christopher Alexander 曾这样解释过模式的概念：“总会有一类问题在我们身边反复出现，模式就是针对这一类问题的通用解法。当问题反复出现时，直接套用这个解法即可，而不需要去重新解决问题。”

> Each pattern describes a problem which occurs over and over again in our environment, and then describes the core of the solution to that problem, in such a way that you can use this solution a million times over, without ever doing it the same way twice.

* 后来，模式的思想也被引入了软件工程领域，[软件工程](https://book.douban.com/subject/6047742/)中提出了以下设计原则，设计模式也遵循了这些原则
  * 开闭原则（The Open-Closed Principle，OCP）：对扩展开放，对修改关闭。修改程序时，不需要修改类内部代码就可以扩展类的功能，如装饰器模式
  * Liskov 替换原则（Liskov Substitution Principle，LSP）：任何基类出现的地方，都可以用派生类替换
  * 依赖倒置原则（Dependency Inversion Principle，DIP）：针对接口（纯虚函数）编程，而非针对实现编程
  * 接口分离原则（Interface Segregation Principle，ISP）：接口功能的粒度应该尽可能小，多个功能分离的小接口、单个合并了这些功能的大接口，前者是更好的做法，因为这样降低了依赖，耦合度更低
  * 发布复用等价性原则（Release Reuse Equivalency Principle，REP）：复用的粒度就是发布的粒度。第三方库的作者需要维护每个版本，作者可以修改代码，但是用户不需要了解源码的变化，而可以自由选择使用哪个版本的库，因此库作者应该将可复用的类打包成包，以包为单位来更新，而不是更新每个类，比如 [Boost](https://www.boost.org/) 有一个版本号，而其中的每个子部分（如 [Boost.Asio](https://www.boost.org/doc/libs/1_66_0/doc/html/boost_asio.html)）又有各自独立的版本号
  * 共同封装原则（Common Closure Principle，CCP）：一同变更的类应该合在一起，如果一些类处理相同的功能或行为域，那么这些类应该根据内聚性分到一组打包，这样需要修改某个域的功能时，只需要修改这个包中的代码
  * 共同复用原则（Common Reuse Principle，CRP）：不能一起被复用的类不能被分到一组。当包中的类变化时，包的版本号也会变化，如果不相关的类被分到一组，就会导致本来无必要的包的版本升级，为此又需要进行本来无必要的集成和测试
* 设计模式是从已有的软件设计中，针对重复出现的问题提取出的一套经验论，其概念源自 *[Design Patterns: Elements of Reusable Object-Oriented Software](https://www.oreilly.com/library/view/design-patterns-elements/0201633612/)*，此书由四人合著，因此简称 GoF（Gang of Four）。GoF 总结归纳了 23 种设计模式，分为创建型（Creational）、结构型（Structural）、行为型（Behavioral）三类。一些设计模式已经用在编程语言或框架中（如 C#、Qt、RxJS）。设计模式减少了术语交流的沟通成本，灵活使用它来写出低耦合少重复的代码。

|创建型模式|中文名|说明|实现|
|:-|:-|:-|:-|
|Abstract Factory/Kit|抽象工厂模式|[README](docs/Creational_Pattern/Abstract_Factory.md)|[C++](src/Abstract_Factory.cpp)|
|Builder|建造者模式|[README](docs/Creational_Pattern/Builder.md)|[C++](src/Builder.cpp)|
|Factory Method/Virutal Contructor|工厂方法模式|[README](docs/Creational_Pattern/Factory_Method.md)|[C++](src/Factory_Method.cpp)|
|Prototype|原型模式|[README](docs/Creational_Pattern/Prototype.md)|[C++](src/Prototype.cpp)|
|Singleton|单例模式|[README](docs/Creational_Pattern/Singleton.md)|[C++](src/Singleton.cpp)|

|结构型模式|中文名|说明|实现|
|:-|:-|:-|:-|
|Adapter/Wrapper|适配器模式|[README](docs/Structural_Pattern/Adapter.md)|[C++](src/Adapter.cpp)|
|Bridge/Handle/Body|桥接模式|[README](docs/Structural_Pattern/Bridge.md)|[C++](src/Bridge.cpp)|
|Composite|组合模式|[README](docs/Structural_Pattern/Composite.md)|[C++](src/Composite.cpp)|
|Decorator/Wrapper|装饰器模式|[README](docs/Structural_Pattern/Decorator.md)|[C++](src/Decorator.cpp)|
|Facade|外观模式|[README](docs/Structural_Pattern/Facade.md)|[C++](src/Facade.cpp)|
|Flyweight|享元模式|[README](docs/Structural_Pattern/Flyweight.md)|[C++](src/Flyweight.cpp)|
|Proxy/Surrogate|代理模式|[README](docs/Structural_Pattern/Proxy.md)|[C++](src/Proxy.cpp)|

|行为型模式|中文名|说明|实现|
|:-|:-|:-|:-|
|Chain of Responsibility|责任链模式|[README](docs/Behavioral_Pattern/Chain_of_Responsibility.md)|[C++](src/Chain_of_Responsibility.cpp)|
|Command/Action/Transaction|命令模式|[README](docs/Behavioral_Pattern/Command.md)|[C++](src/Command.cpp)|
|Interpreter|解释器模式|[README](docs/Behavioral_Pattern/Interpreter.md)|[C++](src/Interpreter.cpp)|
|Iterator/Cursor|迭代器模式|[README](docs/Behavioral_Pattern/Iterator.md)|[C++](src/Iterator.cpp)|
|Mediator|中介者模式|[README](docs/Behavioral_Pattern/Mediator.md)|[C++](src/Mediator.cpp)|
|Memento/Token|备忘录模式|[README](docs/Behavioral_Pattern/Memento.md)|[C++](src/Memento.cpp)|
|Observer/Dependents/Publish-Subscribe|观察者模式|[README](docs/Behavioral_Pattern/Observer.md)|[C++](src/Observer.cpp)|
|State/Objects for States|状态模式|[README](docs/Behavioral_Pattern/State.md)|[C++](src/State.cpp)|
|Strategy/Policy|策略模式|[README](docs/Behavioral_Pattern/Strategy.md)|[C++](src/Strategy.cpp)|
|Template Method|模板方法模式|[README](docs/Behavioral_Pattern/Template_Method.md)|[C++](src/Template_Method.cpp)|
|Visitor|访问者模式|[README](docs/Behavioral_Pattern/Visitor.md)|[C++](src/Visitor.cpp)|
