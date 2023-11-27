# 学习虚拟线程原理

## 概述

虚拟线程(VirtualThread)的根基是 Continuation，这个词翻译是延续，我自己的理解在编程语境中是中断恢复的意思。
如果想了解 VirtualThread 直接查看 JDK 源码即可，而 Continuation 的具体实现都在 VM 中。我想深入了解一下 Continuation 在 vm
中的具体实现。

## 例子 {collapsible="true"}

```Java
public class ContinuationTest {
    public static void main(String[] args) {
        ContinuationScope scope = new ContinuationScope("scope");
        Continuation continuation = new Continuation(scope, () -> {
            System.out.println("Running before yield");
            Continuation.yield(scope);
            System.out.println("Running after yield");
            Continuation.yield(scope);
        });
        System.out.println("First run");
        while (!continuation.isDone()) {
            continuation.run();
            System.out.println("Second run");
        }
        System.out.println("Done");
    }
}
```

运行结果:

```text
First run
Running before yield
Second run
Running after yield
Second run
Second run
Done
```

## 解析 yield 方法的 jdk 实现

> //........... 表示该部分的代码省略了，目的是为了把关注点放在关键方法上

```Java
public class Continuation {
    public static boolean yield(ContinuationScope scope) {
        Continuation cont = JLA.getContinuation(currentCarrierThread());
        //.........
        return cont.yield0(scope, null);
    }
}
```

```java
public class Continuation {
    private boolean yield0(ContinuationScope scope, Continuation child) {
        //.........
        int res = doYield();
        //..........
        if (res == 0)
            onContinue(); // 恢复执行
        else
            onPinned0(res); // 恢复失败，抛出异常
        return res == 0;
    }

    private native static int doYield();

    protected void onContinue() {
    }
    private void onPinned0(int reason) {
        onPinned(pinnedReason(reason));
    }
    protected void onPinned(Pinned reason) {
        throw new IllegalStateException("Pinned: " + reason);
    }
}
```

根据上面的方法内容得知，关键是这个 doYield 的 native 调用，在此处完成了 continuation 的挂起

## 解析 native 方法 doYield

虽然 doYield 是个 native 方法，执行的是 Jni 调用，理应在 vm 源码找到对应的 C/C++ 方法。但 vm
为了提高执行效率，将该方法编码成了机器码。生成机器码的方法如下(x86_64)：

```c++
// sharedRuntime_x86_64.cpp
static void gen_continuation_yield(MacroAssembler* masm,
                                   const VMRegPair* regs,
                                   OopMapSet* oop_maps,
                                   int& frame_complete,
                                   int& stack_slots,
                                   int& compiled_entry_offset) {
    //.............                                   
    // r15_thread 对应的是 x86 中的 r15 寄存器，vm 用来储存当前线程对象在堆内的地址
    // movptr 与 x86 汇编中的 move 一样，将 r15 寄存器的值复制到 c_rarg0 (rdi) 寄存器中
    __ movptr(c_rarg0, r15_thread);
    // rsp 寄存器是栈寄存器，记录栈顶的地址
    // 将 rsp 寄存器的值保存到 c_rarg1 (rsi) 寄存器中
    __ movptr(c_rarg1, rsp);
    // 调用 vm 的方法 freeze_entry()
    __ call_VM_leaf(Continuation::freeze_entry(), 2);
    //............
}
```

```c++
static address freeze_entry = nullptr;
address Continuation::freeze_entry() {
    // 返回 freeze_entry 方法的地址
    return ::freeze_entry;
}
```

### continuation 的 freeze 流程

从上面的调用可知，yield 方法的会调用 freeze（冻结） 方法，所以 vm 官方是把 yield 要做的事情抽象成了“冻结”的概念。
freeze_entry 的方法内容如下:
> 可以先不去深究 JRT_BLOCK_ENTRY 这个宏的作用，关注方法内容即可。

```c++
template<typename ConfigT>
static JRT_BLOCK_ENTRY(int, freeze(JavaThread* current, intptr_t* sp))
    // 条件一：快速路径过小(因为栈是从高地址向低地址增长)。条件二：快速路径超过了指定路径（sp是栈顶指针）
    if (current->raw_cont_fastpath() > current->last_continuation()->entry_sp() || current->raw_cont_fastpath() < sp) {
        // 如果不满足快速路径的条件，则设置失效
        current->set_cont_fastpath(nullptr);
    }
    // 执行冻结
    return ConfigT::freeze(current, sp);
JRT_END
```

> 后面肯定有逻辑会涉及到 fastpath，先跳过

```c++
class Config {
public:
    static int freeze(JavaThread* thread, intptr_t* const sp) {
        // 调用了下面的静态方法
        return freeze_internal<SelfT>(thread, sp);
    }
}
```

下面方法被上面的方法调用

```c++
// continuationFreezeThaw.cpp
template<typename ConfigT>
static inline int freeze_internal(JavaThread* current, intptr_t* const sp) {
    ContinuationEntry* entry = current->last_continuation();
    oop oopCont = entry->cont_oop(current);
    //..........
    ContinuationWrapper cont(current, oopCont);
    //..........
    Freeze<ConfigT> freeze(current, cont, sp);
    //..........
    // UseContinuationFastPath：jvm 参数是否启用快速路径,默认是 true
    // current->cont_fastpath(): 上面的 freeze 方法涉及到了 set_cont_fastpath(nullptr)，用这个方式判断是否支持 fastpath
    bool fast = UseContinuationFastPath && current->cont_fastpath();
    // freeze.size_if_fast_freeze_available(): 应该是判断当前的 chunk 是否有足够的空间可以冻结
    if (fast && freeze.size_if_fast_freeze_available() > 0) {
        freeze.freeze_fast_existing_chunk();
        //...........
        return 0;
    }
    //...........
    // 满足 fast 就执行 fast，否则 slow
    freeze_result res = fast ? freeze.try_freeze_fast() : freeze.freeze_slow();
    //..........
    // 收尾工作
    cont.done();
    return res;
}
```

> 在 globals.hpp 中 `develop(bool, UseContinuationFastPath, true, "Use fast-path frame walking in continuations")`
> UseContinuationFastPath 的 默认值是 ture

## freeze 的 slowPath 流程

之前看过 Continuation 的相关介绍，首次 yield 需要复制完整栈帧到堆内存中，之后由于懒加载策略的存在就不会出现全部复制的情况，所以我认为复制全部栈帧的过程对应的就是
slowPath。优先查看 slow 是因为最好按照执行的先后顺序来查看源码，这样比较容易理解。

```c++
NOINLINE freeze_result FreezeBase::freeze_slow() {
    //.........
    // 栈中最后一个 java 方法的栈帧信息
    frame f = freeze_start_frame();
    //.........
    frame caller;
    // 递归冻结
    freeze_result res = recurse_freeze(f, caller, 0, false, true);
    if (res == freeze_ok) {
        finish_freeze(f, caller);
        _cont.write();
    }
    return res;
}
```

### 关于 start frame {collapsible="true"}

```c++
frame FreezeBase::freeze_start_frame() {
    // 栈中最新的帧
    frame f = _thread->last_frame();
    if (LIKELY(!_preempt)) {
        // 当前抢断标记是 false
        return freeze_start_frame_yield_stub(f);
    }
    // 当前正在被抢断
    return freeze_start_frame_safepoint_stub(f);
}
```

可以先不考虑抢断与否，看下 last_frame()

```c++
class JavaThread: public Thread {
    frame last_frame() {
        //............
        return pd_last_frame();
    }
}
```

继续看调用

```c++
class JavaThread: public Thread {
    // 当前java栈帧，以及状态的封装对象
    JavaFrameAnchor _anchor; // Encapsulation of current java frame and it state
}
frame JavaThread::pd_last_frame() {
    // 返回一个栈帧结构体
    return frame(_anchor.last_Java_sp(), _anchor.last_Java_fp(), _anchor.last_Java_pc()); 
}
```

关于 JavaFrameAnchor 的内容，接着看

```c++
class JavaFrameAnchor {
    // 以下是官翻，但该字段的作用是栈指针的值
    // 无论何时_last_Java_sp不为空，其他锚点字段必须有效！ 
    // 堆栈可能不可遍历[用walkable()检查]，但值必须有效。 
    // 分析器显然依赖于此
    intptr_t* volatile _last_Java_sp;
    // 以下是官翻，该字段的作用是java的指令计数器，存储下一个字节码地址
    // 无论何时我们从Java调用本地代码，我们都不能保证组成last_Java_frame的返回地址会在一个可访问的位置，
    // 所以从Java到本地的调用会把那个pc（或者一个足够好的pc来定位oopmap）存储在帧锚中。
    // 由于从Java到本地的调用的帧从不被去优化，我们从不需要修补pc，所以这是可以接受的。
    volatile  address _last_Java_pc;
    // fp 的值和 sp 有关
    intptr_t* volatile        _last_Java_fp;
}
```

### recurse freeze 递归冻结

```c++
NOINLINE freeze_result FreezeBase::recurse_freeze(frame& f, frame& caller, int callee_argsize, bool callee_interpreted, bool top) {
    if (f.is_compiled_frame()) {
        //....................
        // 已经被 jit 优化
        return recurse_freeze_compiled_frame(f, caller, callee_argsize, callee_interpreted);
    } else if (f.is_interpreted_frame()) {
        //....................
        // 解释执行的帧(java帧)
        return recurse_freeze_interpreted_frame(f, caller, callee_argsize, callee_interpreted);
    } else if (_preempt && top && ContinuationHelper::Frame::is_stub(f.cb())) {
        // 被优化过的帧
        return recurse_freeze_stub_frame(f, caller);
    } else {
        // native 方法，
        return freeze_pinned_native;
    }
}
```

此时有三个分支，分别是 jit优化后的帧，解释执行的帧，优化后的帧。而需要优先分析的帧是解释执行的帧和优化后的帧。

#### 解释执行的帧

```c++
NOINLINE freeze_result FreezeBase::recurse_freeze_interpreted_frame(frame& f, frame& caller,
                                                                    int callee_argsize /* incl. metadata */,
                                                                    bool callee_interpreted) {
    // 参数解析
    // f: 当前栈帧中最后一个 java 栈帧
    // caller: 一个栈上分配对象
    // callee_argsize: 默认是0
    // callee_interpreted: 在当前上下文中，是 true
    
    // 栈顶指的是最低的地址, 获取当q方法栈的顶部
    intptr_t* const stack_frame_top = ContinuationHelper::InterpretedFrame::frame_top(f, callee_argsize, callee_interpreted);
    // 获取方法方法栈的底部
    intptr_t* const stack_frame_bottom = ContinuationHelper::InterpretedFrame::frame_bottom(f);
    // bottom 是高地址，所以相减是正数。此处表示当前栈的空间大小是多少字节
    const int fsize = stack_frame_bottom - stack_frame_top;
    
    freeze_result result = recurse_freeze_java_frame<ContinuationHelper::InterpretedFrame>(f, caller, fsize, argsize);
    if (UNLIKELY(result > freeze_ok_bottom)) {
        return result;
    }
    // ..............
    return freeze_ok; 
}
```

由于 recurse_freeze_java_frame 这个方法比较关键，决定了真正的返回值，所以需要重点看下。

```c++
template<typename FKind>
inline freeze_result FreezeBase::recurse_freeze_java_frame(const frame& f, frame& caller, int fsize, int argsize) {
    // 参数解析
    // f:
    // caller:
    // fsize: 当前方法栈的大小
    // argsize:
    
    // 方法栈低和 FreezeBase(组合了 Continuation 的对象)
    if (FKind::frame_bottom(f) >= _bottom_address - 1) {
    } else {
        frame senderf = sender<FKind>(f);
        // 在本文上面涉及到了 recurse_freeze 方法，现在又遇到，说明是个递归调用。
        freeze_result result = recurse_freeze(senderf, caller, argsize, FKind::interpreted, false); // recursive call
        return result;
    }
}
```