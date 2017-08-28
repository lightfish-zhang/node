# node学习笔记——关于V8

## 概念

V8 是Google开发的JavaScript引擎，提供JavaScript运行环境，可以说它就是 Node.js 的发动机。

## 重要知识点

### js引擎的工作流程

- 现在专业的js引擎的工作过程大概是: 源代码->抽象语法树->字节码->JIT->本地代码。
- v8的特点做法是，直接的将抽象语法树通过 JIT 技术转换成本地代码，放弃了在字节码阶段可以进行的一些性能优化，但保证了执行速度。在 V8 生成本地代码后，也会通过 Profiler 采集一些信息，来优化本地代码。虽然，少了生成字节码这一阶段的性能优化， 但极大减少了转换时间。


### 对象、作用域与垃圾回收

- 在v8源码中，`include/v8.h`定义里模板类`class Local`，代码中有注释，笔者添加中文翻译

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
    （闭包）

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

- 上面提到一个javascript的高级特性，闭包，实现外部作用域访问内部作用域中变量的方法。
    + 通过函数式编程，高阶函数返回一个函数对象，该函数引用到高阶函数中声明的变量，外部作用域中，可以通过这个中间函数访问或修改高阶函数的变量。
    + 它的问题在于，一旦有变量引用这个中间函数，这个中间函数将不会释放，高阶函数这个原始作用域也不会释放，作用于中产生的内存占用就不会得到释放，除非不再有引用，才会逐步释放。

#### 对象的操作符重载

- 得益于C++的面向对象特性的强大，可以重载对象的操作符，v8是使用C++编写的引擎，自然使用这个特性来实现js的一些语法
- 例子，javascript是使用“＋”来实现字符串对象的拼接，v8中字符串对象的代码在`v8/src/inspector/string-16.h`，下面贴出部分代码，加上笔者的注释

```cpp
class String16 {
 public:

    //explicit 避免隐式转换
  explicit String16(const std::basic_string<UChar>& impl) : m_impl(impl) {}

    // 重载操作符=，传入常量的对象的引用
  String16& operator=(const String16& other) {
    m_impl = other.m_impl;
    hash_code = other.hash_code;
    return *this;
  }

    // 重载操作符=，转移&&, 传入String16对象的右值
  String16& operator=(String16&& other) {
      // 左值引用转换为右值引用
    m_impl = std::move(other.m_impl);
    hash_code = other.hash_code;
    return *this;
  }

    // 重载操作符+，右操作数是String16对象引用
  inline String16 operator+(const String16& other) const {
    return String16(m_impl + other.m_impl);
  }
    // 重载操作符+, 左操作数是char*字符指针，右操作数是String16对象引用
  inline String16 operator+(const char* a, const String16& b) {
    return String16(a) + b;
  }

 private:
  std::basic_string<UChar> m_impl; // 私有m_impl实质上使用C++标准库的string对象，string对象的操作符+也是被重载过的
}

```

- 题外话，v8的字符串对象的操作，对于前端项目的平常需求是满足的，但是对于node而言，高并发、大流量的网络字节的处理，是使用stream与Buffer来处理的，而Buffer对象的内存分配不是在v8的堆内存中，而是Node在C++层面实现的，使用slab分配机制，笔者以后详细分析。

### 虚拟机

- `Isolate`，一个 Isolate 是一个独立的虚拟机。对应一个或多个线程。但同一时刻 只能被一个线程进入。所有的 Isolate 彼此之间是完全隔离的, 它们不能够有任何共享的资源。如果不显示创建 Isolate, 会自动创建一个默认的 Isolate。

> An isolate is a VM instance with its own heap. It represents an isolated instance of the V8 engine. V8 isolates have completely separate states. Objects from one isolate must not be used in other isolates.

- 上文中提到的`Handle`, `Scope`, `Context`都是在`Isolate`内部的
- 源码`v8/src/isolate.h`与`v8/src/isolate.cc`，以下下贴出笔者觉得有意思的部分代码

```cpp
class Isolate {

  // True if at least one thread Enter'ed this isolate.
  // 线程进入前调用，确认能否进入
  bool IsInUse() { return entry_stack_ != NULL; }

  // Access to top context (where the current function object was created).
  // 返回当前上下文context，当function创建时，需要使用context
  Context* context() { return thread_local_top_.context_; }

  // Returns the global object of the current context. It could be
  // a builtin object, or a JS global object.
  // 获取全局对象
  inline Handle<JSGlobalObject> global_object();

  // Promise的相关函数
  // Push and pop a promise and the current try-catch handler.
  void PushPromise(Handle<JSObject> promise);
  void PopPromise();

  // Return the relevant Promise that a throw/rejection pertains to, based
  // on the contents of the Promise stack
  Handle<Object> GetPromiseOnStackOnThrow();

  // Heuristically guess whether a Promise is handled by user catch handler
  bool PromiseHasUserDefinedRejectHandler(Handle<Object> promise);

}
```

- 将以上代码的promise相关部分展开看，这部分应该和node的线程库Libuv有关联

```cpp
void Isolate::PushPromise(Handle<JSObject> promise) {
  ThreadLocalTop* tltop = thread_local_top();
  PromiseOnStack* prev = tltop->promise_on_stack_;
  Handle<JSObject> global_promise = global_handles()->Create(*promise);
  tltop->promise_on_stack_ = new PromiseOnStack(global_promise, prev);
}


void Isolate::PopPromise() {
  ThreadLocalTop* tltop = thread_local_top();
  if (tltop->promise_on_stack_ == NULL) return;
  PromiseOnStack* prev = tltop->promise_on_stack_->prev();
  Handle<Object> global_promise = tltop->promise_on_stack_->promise();
  delete tltop->promise_on_stack_;
  tltop->promise_on_stack_ = prev;
  global_handles()->Destroy(global_promise.location());
}
```
