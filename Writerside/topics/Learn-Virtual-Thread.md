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

freeze_entry 的方法内容如下，可以先不去深究 JRT_BLOCK_ENTRY 这个宏的作用，关注方法内容即可。

```c++
template<typename ConfigT>
static JRT_BLOCK_ENTRY(int, freeze(JavaThread* current, intptr_t* sp))
    if (current->raw_cont_fastpath() > current->last_continuation()->entry_sp() || current->raw_cont_fastpath() < sp) {
        current->set_cont_fastpath(nullptr);
    }
    return ConfigT::freeze(current, sp);
JRT_END
```