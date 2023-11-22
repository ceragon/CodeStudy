# 学习虚拟线程原理

## 概述

虚拟线程(VirtualThread)的根基是 Continuation，这个词翻译是延续，我自己的理解在编程语境中是中断恢复的意思。
如果想了解 VirtualThread 直接查看 JDK 源码即可，而 Continuation 的具体实现都在 VM 中。我想深入了解一下 Continuation 在 vm
中的具体实现。

## 例子

```Java
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
```