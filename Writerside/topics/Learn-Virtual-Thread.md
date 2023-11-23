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

