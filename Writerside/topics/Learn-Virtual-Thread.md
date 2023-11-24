# 学习虚拟线程原理

## 概述

虚拟线程(VirtualThread)的根基是 Continuation，这个词翻译是延续，我自己的理解在编程语境中是中断恢复的意思。
如果想了解 VirtualThread 直接查看 JDK 源码即可，而 Continuation 的具体实现都在 VM 中。我想深入了解一下 Continuation 在 vm
中的具体实现。

## 例子

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

### continuation 的 slowPath

之前看过 Continuation 的相关介绍，首次 yield 需要复制完整栈帧到堆内存中，之后由于懒加载策略的存在就不会出现全部复制的情况，所以我认为复制全部栈帧的过程对应的就是
slowPath。优先查看 slow 是因为最好按照执行的先后顺序来查看源码，这样比较容易理解。

```c++
NOINLINE freeze_result FreezeBase::freeze_slow() {
    
}
```