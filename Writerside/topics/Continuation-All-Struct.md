# Continuation 涉及的各种结构体

## Continuation

### Continuation java 结构体

```java
package jdk.internal.vm;

public class Continuation {
    private final Runnable target;
    private final ContinuationScope scope;
    private Continuation parent;
    private Continuation child;
    private StackChunk tail;
    private boolean done;
    private volatile boolean mounted;
    private Object yieldInfo;
    private boolean preempted;

    private Object[] scopedValueCache;
}
```

### Continuation vm 结构体

```c++
class jdk_internal_vm_Continuation: AllStatic {
    oop* target;
    ContinuationScope* scope;
    Continuation* parent;
    Continuation* tail;
    bool done;
    bool mounted;
    oop* yieldInfo;
    bool preempted;
}
```

## StackChunk

分为 java 里的结构体和 jvm 中 C++ 结构体

### StackChunk java 结构体

```java
package jdk.internal.vm;

public final class StackChunk {
    private StackChunk parent;
    private int size;
    private int sp;
    private int argsize;
}
```

### jvm 结构体

由于 vm 并没有实际定义这样一个结构体，所以还原了一下

```c++
class jdk_internal_vm_StackChunk: AllStatic {
private:
    StackChunk* parent;
    int size;
    int sp;
    int argsize;
    Continuation* count;
    uint8_t flags;
    address pc;
    int maxThawingSize;
    //-------------------------
    byte stack[];
    //-------------------------
    byte gc_data[];
}
```

#### size 字段的赋值

```c++
class StackChunkAllocator : public MemAllocator {
    const size_t                                 _stack_size;
    virtual oop initialize(HeapWord* mem) const override {
        // hs 的值是 2 
        const size_t hs = oopDesc::header_size();
        // 将 mem + 2 后面的字节全部置0        
        Copy::fill_to_aligned_words(mem + hs, vmClasses::StackChunk_klass()->size_helper() - hs);
        // size 和 sp 都是栈大小。
        // 结合下文，栈大小是需要存储的栈的大小，单位是 word_size(8字节)
        jdk_internal_vm_StackChunk::set_size(mem, (int)_stack_size);
        jdk_internal_vm_StackChunk::set_sp(mem, (int)_stack_size);
    }
public:    
     StackChunkAllocator(Klass* klass,
                      size_t word_size,
                      Thread* thread,
                      size_t stack_size,
                      ContinuationWrapper& continuation_wrapper,
                      JvmtiSampledObjectAllocEventCollector* jvmti_event_collector)
    : MemAllocator(klass, word_size, thread),
    // 设置栈大小
      _stack_size(stack_size),
      _continuation_wrapper(continuation_wrapper),
      _jvmti_event_collector(jvmti_event_collector),
      _took_slow_path(false) {}
}
```

```c++
template <typename ConfigT>
stackChunkOop Freeze<ConfigT>::allocate_chunk(size_t stack_size) {
    InstanceStackChunkKlass* klass = InstanceStackChunkKlass::cast(vmClasses::StackChunk_klass());
    // 计算出 StackChunk 的总大小
    size_t size_in_words = klass->instance_size(stack_size);
    //.............
    // 创建一个分配器
    StackChunkAllocator allocator(klass, size_in_words, current, stack_size, _cont, _jvmti_event_collector);
    // 分配一个块
    stackChunkOop chunk = allocator.allocate();
    return chunk;
}
```

```c++
inline size_t InstanceStackChunkKlass::instance_size(size_t stack_size_in_words) const {
    // StackChunk 由三部分组成
    // - StackChunk 的本体
    // - 存储栈上的内容
    // - gc 数据
    return align_object_size(size_helper() + stack_size_in_words + gc_data_size(stack_size_in_words));
}
```

#### oopDesc::header_size

```c++
class oopDesc {
    volatile markWord _mark;
    union _metadata {
        Klass*      _klass;
        narrowKlass _compressed_klass;
    } _metadata;
    
    static constexpr int header_size() {
        // sizeof(oopDesc): 由两部分组成，mark 和 _metadata指针。不考虑压缩指针的情况，值是 8 + 8 = 16 字节。
        // HeapWordSize: vm 中指针的字节数。不开启压缩指针是 8 字节
        // 最终值是 2
        return sizeof(oopDesc)/HeapWordSize; 
    }   
}
```

