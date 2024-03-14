单例模式是一种设计模式，保证一个类仅有一个实例，并提供一个访问它的全局访问点，该实例被所有程序模块共享。从并发访问的角度思考单例模式中的这个唯一实例，把它视为多个线程都要访问的共享数据。现在考虑这样一种情况：我们无需担心实例初始化之后的线程安全问题，这可能是因为实例一旦创建就是只读的，也可能是初始化之后的操作已经施加了必要的同步操作。基于这样一种简化的情况，专注于实现单例模式的关键：如何线程安全的初始化共享数据。

单例模式中，对实例的初始化2种方式

1.  延迟初始化（lazy initialization）： 也就是通常说的懒汉式。等到必要时才创建单例类的实例（懒的建 QVQ），如果该实例从未被使用，则不会创建 。这种方式的好处是可以减少资源消耗和提高性能，特别是有些实例的创建代价可能不菲，比如创建一个数据库连接或一个线程池。
1.  立即初始化（eager initialization）: 饿汉式。指在程序启动或类加载到内存时立即创建单例类的实例（饿了快吃 QVQ）。这种方式的好处是线程安全，由于单例实例是在类加载时创建的，这通常由类加载器保证，因此不需要额外的同步机制来保证线程安全，实现起来也比较简单。此外，因为实例一开始就创建好了，因此线程需要用到实例时，可以直接获取到实例。

# 1. 立即初始化

```cpp
class Singleton {
 public:
  Singleton(const Singleton&) = delete;
  Singleton& operator=(const Singleton&) = delete;
  Singleton(Singleton&&) = delete;
  Singleton& operator=(Singleton&&) = delete;

  static Singleton& get_instance() { return instance; }

 private:
  Singleton() = default;
  static Singleton instance;
};

Singleton Singleton::instance;
```

`Singleton` 类的 `instance` 是一个静态数据成员，并且在类外进行初始化。这意味着 `instance` 会在程序启动阶段（具体是在进入 `main` 函数之前的静态初始化阶段）就被创建。这种方式被称为饿汉模式，因为它确保了单例实例在程序启动时立即被初始化，无论之后是否会用到这个实例。

饿汉模式的特点：

-   确保了线程安全，因为静态初始化在C++中是线程安全的。
-   可能会增加程序启动时的负担，因为不管实例是否会被用到，它都会被创建

# 2. 延迟初始化

## 2.1. 优雅的local static

*effective c++* 的item4中提到过用静态局部变量 (local static) 消除2个对象间的初始化依赖，保证一个对象在访问另一个对象的时候该对象已经初始化。该方法是基于这样一个事实：控制流程第一次遇到静态数据的声明语句时，数据即进行初始化。

要通过该方式实现线程安全的单例模式，光基于这一个事实可不够，还需要基于这样一个事实： 在C++11之后，规定静态数据的初始化只在某一个线程上单独进行，在初始化完成之前，其他线程不会越过静态数据的声明而继续运行。

为什么需要这样一个事实呢，因为在C++11之前，把局部变量声明为静态数据并不是线程安全的，是会导致数据竞争的。多个线程调用都可能到达静态数据的声明处，而这些线程可能都认为自己是第一个到达的线程，从而试图初始化静态数据，这肯定是要造成数据错乱的。不仅如此，也可能是某个线程正在初始化静态数据，还没完成，此时另一个线程却试图使用它，此时使用的数据肯定也是不完整的。

基于上面2个事实，可用静态局部变量优雅的实现线程安全的单例模式：

```cpp
class Singleton {
 public:
  Singleton(const Singleton&) = delete;
  Singleton& operator=(const Singleton&) = delete;
  Singleton(Singleton&&) = delete;
  Singleton& operator=(Singleton&&) = delete;

  static Singleton& get_instance() {
    static Singleton instance;
    return instance;
  }

 private:
  Singleton() = default;
};
```

`Singleton` 类的 `instance` 是一个局部静态变量，它在 `get_instance()` 函数中定义。这种方式被称为懒汉模式，因为它延迟了单例实例的创建时机，只有在第一次调用 `get_instance()` 时才会创建实例。
懒汉模式的特点：

-   保证了线程安全，C++11及以后的标准规定，局部静态变量的初始化是线程安全的。
-   实例的创建被延迟到了首次需要时，这可以减少不必要的资源占用，特别是当实例的创建代价较大或程序可能不会使用到这个实例时

## 2.2. 标准库call_once和call_flag

在多线程环境下，`std::call_once` 和 `std::once_flag` 配合使用来确保某个函数（如初始化函数）只被执行一次，即使在并发环境中也是如此。这是通过以下方式实现的：

-   `std::once_flag`：这是一个标记，用来记录是否已经调用了一次函数。
-   `std::call_once`：这个函数接收一个 `std::once_flag` 和一个要调用的函数。它检查标记，如果指定的函数尚未被调用，`std::call_once` 会执行它。如果函数已经被调用，`std::call_once` 将不会再次执行该函数。

结合这2个函数，我们可以确保实例在某一线程中唯一初始化。

```cpp
#include <mutex>

class Singleton {
 public:
  Singleton(const Singleton&) = delete;
  Singleton& operator=(const Singleton&) = delete;
  Singleton(Singleton&&) = delete;
  Singleton& operator=(Singleton&&) = delete;

  static Singleton& get_instance() {
    std::call_once(init_flag, &Singleton::init);
    return *instance_ptr;
  }

 private:
  Singleton() = default;
  static std::once_flag init_flag;
  static Singleton* instance_ptr;
  static void init() { instance_ptr = new Singleton(); }
};
```

引用：

1.  [C++单例模式 --- 知乎](https://zhuanlan.zhihu.com/p/37469260)
1.  *effective c++* item4
1.  *C++并发编程第二版*  3.3
1.  *linux多线程服务端编程* 2.5



Reference:

