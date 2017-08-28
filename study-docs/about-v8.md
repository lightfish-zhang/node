# node学习笔记——关于V8

## 概念

V8 是Google开发的JavaScript引擎，提供JavaScript运行环境，可以说它就是 Node.js 的发动机。

## 重要知识点

### js引擎的工作流程

- 现在专业的js引擎的工作过程大概是: 源代码->抽象语法树->字节码->JIT->本地代码。
- v8的特点做法是，直接的将抽象语法树通过 JIT 技术转换成本地代码，放弃了在字节码阶段可以进行的一些性能优化，但保证了执行速度。在 V8 生成本地代码后，也会通过 Profiler 采集一些信息，来优化本地代码。虽然，少了生成字节码这一阶段的性能优化， 但极大减少了转换时间。


### 对象、作用域与垃圾回收

- 在v8源码中，`include/v8.h`定义里模板类`class Local`，代码中有注释，做翻译解释如下

```cpp
/**
An object reference managed by the v8 garbage collector.
    对象引用由v8垃圾回收管理

All objects returned from v8 have to be tracked by the garbage
collector so that it knows that the objects are still alive.
    所有从v8返回的对象，都被垃圾回收器追踪与知道对象是否活跃。

Also, because the garbage collector may move objects, it is unsafe to
point directly to an object.
    因为垃圾回收器可能会回收这个对象，如果使用指针直接指向一个对象的地址，那是不安全的。

Instead, all objects are stored in
handles which are known by the garbage collector and updated
whenever an object moves.  Handles should always be passed by value
(except in cases like out-parameters) and they should never be
allocated on the heap.
    所以，所有对象都存储在handles中，handle是被垃圾回收器管理，如果对象被回收就会更新handle.
    Handles一般是用作传参（在输出参数情况下除外），它们不应该被分配在堆heap中

There are two types of handles: local and persistent handles.
    handles有两种，Local 和 Persistent （局部的 与 持久的）

Local handles are light-weight and transient and typically used in
local operations.
    Local handles是轻量的，短暂使用的，一般用于局部的运算。（local本意是局部）

They are managed by HandleScopes. That means that a
HandleScope must exist on the stack when they are created and that they are
only valid inside of the HandleScope active during their creation.
    Local被HandleScopes管理，一个HandleScopes应该存储在栈stack中，当Local被创建后，
    当HandleScopes处于活跃期间，Local才是合法存在的（也就是说，HandleScopes不再处于活跃，Local对象就会被回收）


For passing a local handle to an outer HandleScope, an EscapableHandleScope
and its Escape() method must be used.
    如果，一个local handle传参到外部的HandleScope或者EscapableHandleScope，必须被使用它的Escape()方法

Persistent handles can be used when storing objects across several
independent operations and have to be explicitly deallocated when they're no
longer used.
    Persistent handles(持久的对象)，可以被用于处理多个独立操作，它如果不在使用，必须被显式回收

It is safe to extract the object stored in the handle by
dereferencing the handle (for instance, to extract the Object* from
a Local<Object>);
    通过封装handle，提取存储在handle的对象，这是安全的。例如，从`Local<Object>`中提取`Object*`指针 ( 这是说把对象的操作等细节封装在Local类中，暴露出来public的函数是安全的 )

the value will still be governed by a handle
behind the scenes and the same rules apply to these values as to
their handles.
    对象的值由handle在背后处理

*/
template <class T>
class Local {
 // .....

 // Handle is an alias for Local for historical reasons.
 // 由于历史原因，Handle是Local的别名
template <class T>
using Handle = Local<T>;
```

#### 对象的操作符重载

- 得益于C++的面向对象特性的强大，可以重载对象的操作符
