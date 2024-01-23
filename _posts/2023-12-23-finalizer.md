
## 1、Finalizer机制

调用时机：在被回收之前
存在的问题：
* 执行不及时，`finalize()`方法是在守护线程中执行，当一个对象变成一个不可达对象后不能及时执行finalize()方法。如果资源没来得及释放会引发问题。
* 不保证执行，如果依赖`finalize()`来更新持久化状态，例如释放数据库的锁、文件的锁则可能使程序陷入死锁。
* 只会执行一次，如果一个不可达对象执行过`finalize()`方法后又变成一个可达对象再次变为不可达对象后不会再执行`finalize()`方法。

**推荐使用Cleaner机制代替Finalizer机制。**

## 2、Finalizer原理
```cpp
art/runtime/mirror/class-alloc-inl.h

template<bool kIsInstrumented, Class::AddFinalizer kAddFinalizer, bool kCheckAddFinalizer>
inline ObjPtr<Object> Class::Alloc(Thread* self, gc::AllocatorType allocator_type) {
  CheckObjectAlloc();
  gc::Heap* heap = Runtime::Current()->GetHeap();
  bool add_finalizer;
  switch (kAddFinalizer) {
    case Class::AddFinalizer::kUseClassTag:
      add_finalizer = IsFinalizable();
      break;
    case Class::AddFinalizer::kNoAddFinalizer:
      add_finalizer = false;
      DCHECK_IMPLIES(kCheckAddFinalizer, !IsFinalizable());
      break;
  }
  // Note that the `this` pointer may be invalidated after the allocation.
  ObjPtr<Object> obj =
      heap->AllocObjectWithAllocator<kIsInstrumented, /*kCheckLargeObject=*/ false>(
          self, this, this->object_size_, allocator_type, VoidFunctor());
  if (add_finalizer && LIKELY(obj != nullptr)) {
    heap->AddFinalizerReference(self, &obj);
    if (UNLIKELY(self->IsExceptionPending())) {
      // Failed to allocate finalizer reference, it means that the whole allocation failed.
      obj = nullptr;
    }
  }
  return obj;
}
```

Java中除了四大引用类型外，虚拟机内部还设计了FinalizerReference类型来支持Finalizer机制。

虚拟机加载过程中会将重写了`Object#finalize()`方法的类型标记为`finalizeable`类型。每次创建`finalizeable`对象时，虚拟机会创建一个`FinalizerReference`引用对象，并将其暂存到一个全局链表中。

heap.cc
```
void Heap::AddFinalizerReference(Thread* self, ObjPtr<mirror::Object>* object) {
    ScopedObjectAccess soa(self);
    ScopedLocalRef<jobject> arg(self->GetJniEnv(), soa.AddLocalReference<jobject>(*object));
    jvalue args[1];
    args[0].l = arg.get();
    // 调用 Java 层静态方法 FinalizerReference#add
    InvokeWithJValues(soa, nullptr, WellKnownClasses::java_lang_ref_FinalizerReference_add, args);
    *object = soa.Decode<mirror::Object>(arg.get());
}
```
FinalizerReference.java
```
// 关联的引用队列
public static final ReferenceQueue<Object> queue = new ReferenceQueue<Object>();
// 全局链表头指针（使用一个双向链表持有 FinalizerReference，否则没有强引用的话引用对象本身直接就被回收了）
private static FinalizerReference<?> head = null;

private FinalizerReference<?> prev;
private FinalizerReference<?> next;

// 从 Native 层调用
public static void add(Object referent) {
    // 创建 FinalizerReference 引用对象，并关联引用队列
    FinalizerReference<?> reference = new FinalizerReference<Object>(referent, queue);
    synchronized (LIST_LOCK) {
        // 头插法加入全局单链表
        reference.prev = null;
        reference.next = head;
        if (head != null) {
            head.prev = reference;
        }
        head = reference;
    }
}

public static void remove(FinalizerReference<?> reference) {
    // 从双向链表中移除，代码略
}

```

# 3、finalze()执行分析

虚拟机启动时会启动一系列守护线程，其中除了引用入队的ReferenceQueueDaemon线程，还包括执行Finalizer机制的FinalizerDaemon线程。FinalizeDaemon线程会轮询观察引用队列，并执行对象的`finalize()`方法。



