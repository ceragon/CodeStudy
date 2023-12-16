# 学习 CMS GC

## 概述

作为经典的垃圾回收器，虽然无法精确控制 STW 的时间，但是 GC 过程对 mutator 线程影响较小，因此吞吐量和内存占用都有明显优势。

通过查看 OpenJDK11 的源码，来一探 CMS 的执行过程。

## 大纲

- cms 的触发条件
- ....

## cms 的触发条件

也就是老年代 gc 的触发条件。cms GC 的第一个阶段是初始标记，且该阶段需要 STW，对应的是以下代码

```c++
class VM_CMS_Initial_Mark: public VM_CMS_Operation {
    virtual void doit();
}
```

而上面的代码会在 VMThread 线程中执行，理由是父类 VM_CMS_Operation 代码如下：

```c++
class VM_CMS_Operation: public VM_Operation {
}
```

### 初始标记的触发条件

初始标记的触发和 VM_CMS_Initial_Mark 类初始化有关，因此反向查看它的调用。以下方法是唯一调用

```c++
void CMSCollector::collect_in_background(GCCause::Cause cause) {
    switch (_collectorState) {
        case InitialMarking:
            {
                VM_CMS_Initial_Mark initial_mark_op(this);
                VMThread::execute(&initial_mark_op);
            }
            break;
    }
}
```

接着查看 collect_in_background 方法的调用

```c++
void ConcurrentMarkSweepThread::run_service() {
    while (!should_terminate()) {
        // 睡眠，并等到下一次 cms 循环开始
        sleepBeforeNextCycle();
        // 被唤醒了，判断是 fullGC 还是 cms gc
        GCCause::Cause cause = _collector->_full_gc_requested ?
            _collector->_full_gc_cause : GCCause::_cms_concurrent_mark;
        // 执行 gc
        _collector->collect_in_background(cause);   
    }
}
```

如果关注点在触发条件的话，和 sleepBeforeNextCycle 有关系

```c++
void ConcurrentMarkSweepThread::sleepBeforeNextCycle() {
    while (!should_terminate()) {
        // 默认值是 2000
        if(CMSWaitDuration >= 0) {
            wait_on_cms_lock_for_scavenge(CMSWaitDuration);
        } else {
            wait_on_cms_lock(CMSCheckInterval);
        }
        if (_collector->shouldConcurrentCollect()) {
            // 条件为真后被唤醒
            return;
        }
    }   
}
```

```c++
bool CMSCollector::shouldConcurrentCollect() {
    if (_full_gc_requested) {
        return true;
    }
    if (!UseCMSInitiatingOccupancyOnly) {
        if (stats().valid()) {
            if (stats().time_until_cms_start() == 0.0) {
                return true;
            }
        } else {
            if (_cmsGen->occupancy() >= _bootstrap_occupancy) {
                return true;
            }
        }
    }
    
    if (_cmsGen->should_concurrent_collect()) {
        return true;
    }
    
    CMSHeap* heap = CMSHeap::heap();
    if (heap->incremental_collection_will_fail(true /* consult_young */)) {
        return true;
    }
    
    if (MetaspaceGC::should_concurrent_collect()) {
        return true;
    }
    if (CMSTriggerInterval >= 0) {
        //TODO:
    }
    return false;
}
```