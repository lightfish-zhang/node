# v8源码研究——链式作用域

## 前言

闲着闲着，找下v8链式作用域的堆栈吧

## 先查资料

- v8的社区的源码分析的文章比较少呢，去stackoverflow看，只有寥寥的一个问答

```
After some debugging, here is the trace of function calls for property access in v8 :

 ic/ic.cc : Load 

     objects.cc : GetProperty

        lookup.cc : GetDataValue

             lookup.cc : FetchValue 

This is the generic flow, based on the kind of properties you are trying to access (for example, array or objects) it may slightly vary.
```

## 链式作用域的查找过程


- 先举个方便理解链式作用域的例子

```js
function f(o) {
  return o.x;
}
```

- 这是个好例子，代码未执行时，你不知道输入参数`o`是什么，如果我不输入，或者输入乱七八糟参数，比如基本类型的数字，js引擎会怎么处理？
- 还有，由于代码未运行，编译器也不知道输入的`o`是啥，那么js的字节码层次上又是如何寻找`property`呢，如果是c/cpp语言，编译时就已经确定结构体成员的偏移量了呢，但是js是怎么做的呢？
- 答案是，v8的`inline caches`，v8的相关代码在`v8/src/ic/ic.cc`

### Inline caches: accelerating unoptimized code

- 有关Inline caches的资料，来自<A tour of V8: full compiler>，伯乐在线有翻译，可惜翻译不对味，本文结尾有链接(笔者也觉得原文很难翻译，有兴趣的直接看原文好了)
- 引用部分要点

>> V8使用IC处理了大量的操作：FC使用IC来实现读取、存储、函数调用、二元运算符、一元运算符、比较运算符以及ToBoolean隐操作符

>> IC的实现称为Stub。Stub在使用层面上像函数：调用、返回。

>> 一旦Stub碰到了优化代码无法解决的操作，它会调用C++运行时代码来进行处理。

```
;; FC调用
ldr   r0, [fp, #+8]     ; 从栈中读取参数”o“
ldr   r2, [pc, #+84]    ; 从固定的位置读取”x“
ldr   ip, [pc, #+84]    ; 从固定位置载入uninitialized态的stub
blx   ip                ; 调用stub
...
dd    0xabcdef01        ; 上面拿到的stub地址
                        ; 当stub出现处理不了的操作时，这里的stub会被换成新的stub

```

- 简单来说，向本文的js例子，就是一段没法直接编译的函数，需要通过`stub`，在字节码中调用c/cpp程序来处理，c/cpp处理完就把结果返回给字节码

### Inline caches与链式作用域

- 看cpp代码, 过程如下
- 先判断作用域的对象是否为Null或undefined，如果是就给js抛出错误，以及`stub`的堆栈处理

```cpp
//  ic/ic.cc : Load 
MaybeHandle<Object> LoadIC::Load(Handle<Object> object, Handle<Name> name) {
  // If the object is undefined or null it's illegal to try to get any
  // of its properties; throw a TypeError in that case.
  if (object->IsNullOrUndefined(isolate())) {
    if (FLAG_use_ic && state() != UNINITIALIZED && state() != PREMONOMORPHIC) {
      // Ensure the IC state progresses.
      TRACE_HANDLER_STATS(isolate(), LoadIC_NonReceiver);
      update_receiver_map(object);
      PatchCache(name, slow_stub());
      TRACE_IC("LoadIC", name);
    }
    return TypeError(MessageTemplate::kNonObjectPropertyLoad, object, name);
  }
```

- 接下来，`ic/ic.cc : Load`调用`ic/ic.cc : LookupForRead`这个模块内静态方法

```cpp
  // Named lookup in the object.
  LookupIterator it(object, name);
  LookupForRead(&it);
```

```cpp
static void LookupForRead(LookupIterator* it) {
 // 这个for循环是链式作用域的一层一层查找吗
  for (; it->IsFound(); it->Next()) {
    switch (it->state()) {
      case LookupIterator::NOT_FOUND:
      case LookupIterator::TRANSITION:
        UNREACHABLE();
      case LookupIterator::JSPROXY:
        return;
      case LookupIterator::INTERCEPTOR: {
        // If there is a getter, return; otherwise loop to perform the lookup.
        Handle<JSObject> holder = it->GetHolder<JSObject>();
        if (!holder->GetNamedInterceptor()->getter()->IsUndefined(
                it->isolate())) {
          return;
        }
        break;
      }
      case LookupIterator::ACCESS_CHECK:
        // ICs know how to perform access checks on global proxies.
        if (it->GetHolder<JSObject>()->IsJSGlobalProxy() && it->HasAccess()) {
          break;
        }
        return;
      case LookupIterator::ACCESSOR:
      case LookupIterator::INTEGER_INDEXED_EXOTIC:
      case LookupIterator::DATA:
        return;
    }
  }
}
```

- 我找到了上面的代码，这是对迭代器`LookupIterator`的迭代操作，我需要找`LookupIterator`的声明与相关函数，代码在`src/lookup.h`与`src/lookup.cc`
- 下面是`LookupIterator`的构造函数，绕的有点多，结合上面的`LookupIterator it(object, name);`来看

```cpp
class V8_EXPORT_PRIVATE LookupIterator final BASE_EMBEDDED {
    /*...*/
  LookupIterator(Handle<Object> receiver, Handle<Name> name,
                 Configuration configuration = DEFAULT)
      : LookupIterator(name->GetIsolate(), receiver, name, configuration) {}

  LookupIterator(Isolate* isolate, Handle<Object> receiver, Handle<Name> name,
                 Configuration configuration = DEFAULT)
      : LookupIterator(isolate, receiver, name, GetRoot(isolate, receiver),
                       configuration) {}

  LookupIterator(Handle<Object> receiver, Handle<Name> name,
                 Handle<JSReceiver> holder,
                 Configuration configuration = DEFAULT)
      : LookupIterator(name->GetIsolate(), receiver, name, holder,
                       configuration) {}

  LookupIterator(Isolate* isolate, Handle<Object> receiver, Handle<Name> name,
                 Handle<JSReceiver> holder,
                 Configuration configuration = DEFAULT)
      : configuration_(ComputeConfiguration(configuration, name)),
        interceptor_state_(InterceptorState::kUninitialized),
        property_details_(PropertyDetails::Empty()),
        isolate_(isolate),
        name_(isolate_->factory()->InternalizeName(name)),
        receiver_(receiver),
        initial_holder_(holder),
        // kMaxUInt32 isn't a valid index.
        index_(kMaxUInt32),
        number_(static_cast<uint32_t>(DescriptorArray::kNotFound)) {
#ifdef DEBUG
    uint32_t index;  // Assert that the name is not an array index.
    DCHECK(!name->AsArrayIndex(&index));
#endif  // DEBUG
    Start<false>();
    /*...*/
```

- `it->state()`就是返回成员`state_`

```cpp
State state() const { return state_; }
```

- 接下来就看`LookupIterator::Next()`

```cpp
void LookupIterator::Next() {
  DCHECK_NE(JSPROXY, state_);
  DCHECK_NE(TRANSITION, state_);
  DisallowHeapAllocation no_gc; // no gc ? 有点意思，以后研究下
  has_property_ = false;

  JSReceiver* holder = *holder_;
  Map* map = holder->map();

  if (map->IsSpecialReceiverMap()) {
    state_ = IsElement() ? LookupInSpecialHolder<true>(map, holder)
                         : LookupInSpecialHolder<false>(map, holder);
    if (IsFound()) return;
  }

  IsElement() ? NextInternal<true>(map, holder)
              : NextInternal<false>(map, holder);
}
```

```cpp
// src/objects-inl.h
inline bool IsSpecialReceiverInstanceType(InstanceType instance_type) {
  return instance_type <= LAST_SPECIAL_RECEIVER_TYPE;
}
```

```cpp
  //  src/objects.h
  // Boundary for testing JSReceivers that need special property lookup handling
  LAST_SPECIAL_RECEIVER_TYPE = JS_SPECIAL_API_OBJECT_TYPE,
```

- 源码里的注释表示，`LAST_SPECIAL_RECEIVER_TYPE`是边界条件，用来检查当前对象是否是其他对象的属性，是否可以lookup到上一个对象
- 笔者想每个对象初始化时都会确认这个类型的值吧，这个对象是否是其他对象的属性

- `LookupIterator::NextInternal()`会调用到`LookupIterator::NextHolder()`

```cpp
JSReceiver* LookupIterator::NextHolder(Map* map) {
  DisallowHeapAllocation no_gc;
  if (map->prototype() == heap()->null_value()) return NULL;
  if (!check_prototype_chain() && !map->has_hidden_prototype()) return NULL;
  return JSReceiver::cast(map->prototype()); // 原型链
}
```

- 笔者懵逼了，好像搞错了什么，作用域链的原理和原型链很类似，笔者搞混了，上面的例子是原型链!!!，算了，接着看下代码

- 找到映射类`Map`了，让我看一下是不是哈希表，源代码文件在`src/objects/map.h`

```cpp
// src/objects.h
void Map::SetPrototype(Handle<Map> map, Handle<Object> prototype,
                       PrototypeOptimizationMode proto_mode) {
  RuntimeCallTimerScope stats_scope(*map, &RuntimeCallStats::Map_SetPrototype);

  bool is_hidden = false;
  if (prototype->IsJSObject()) {
    Handle<JSObject> prototype_jsobj = Handle<JSObject>::cast(prototype);
    JSObject::OptimizeAsPrototype(prototype_jsobj, proto_mode);

    Object* maybe_constructor = prototype_jsobj->map()->GetConstructor();
    if (maybe_constructor->IsJSFunction()) {
      JSFunction* constructor = JSFunction::cast(maybe_constructor);
      Object* data = constructor->shared()->function_data();
      is_hidden = (data->IsFunctionTemplateInfo() &&
                   FunctionTemplateInfo::cast(data)->hidden_prototype()) ||
                  prototype->IsJSGlobalObject();
    } else if (maybe_constructor->IsFunctionTemplateInfo()) {
      is_hidden =
          FunctionTemplateInfo::cast(maybe_constructor)->hidden_prototype() ||
          prototype->IsJSGlobalObject();
    }
  }
  map->set_has_hidden_prototype(is_hidden);

  WriteBarrierMode wb_mode = prototype->IsNull(map->GetIsolate())
                                 ? SKIP_WRITE_BARRIER
                                 : UPDATE_WRITE_BARRIER;
  map->set_prototype(*prototype, wb_mode);
}
```

- 笔者困了，不看了



## 参考资料

- [A tour of V8: full compiler](http://jayconrod.com/posts/51/a-tour-of-v8-full-compiler)