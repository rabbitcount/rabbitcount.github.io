---
title: CMS 垃圾回收启动的时机
date: 2018-02-14 22:54:15
tags:
- jvm
- gc
---

CMS(Concurrent-Mark-Sweep)是 Hotspot 虚拟机中目前使用最广泛的老年代垃圾回收器，在响应时间这个GC 指标上表现优异。CMS 在什么时机开始工作？触发的条件都有哪些？今天我们可以从 OpenJDK 中 CMS 这一块的实现来分析一下，该实现主要位于_concurrentMarkSweepGeneration_ 和 _ConcurrentMarkThread_ 这两个类中。

![](http://anocelot-wiki.oss-cn-hangzhou.aliyuncs.com/java/linux_monitor/image2017-12-25%2014_8_31.png)

## 检查的时机

由 CMS 垃圾回收线程执行垃圾回收的 检查逻辑。

**ConcurrentMarkThread.cpp**
```
void ConcurrentMarkThread::run() {
  // 全局初始化
  while (!_should_terminate) {

    // 一直等待，直到可以开始垃圾回收
    sleepBeforeNextCycle();

    //...
    //...
    // 执行具体的垃圾回收逻辑
    //...
    //...
  }
}
```

_sleepBeforeNextCycle_ 的 实现如下：

**ConcurrentMarkThread.cpp**
```
void ConcurrentMarkSweepThread::sleepBeforeNextCycle() {
  while (!_should_terminate) {
    if (CMSIncrementalMode) {
      icms_wait();
      if(CMSWaitDuration >= 0) {
        wait_on_cms_lock_for_scavenge(CMSWaitDuration);
      }
      return;
    } else {
      if(CMSWaitDuration >= 0) {
        wait_on_cms_lock_for_scavenge(CMSWaitDuration);
      } else {
        wait_on_cms_lock(CMSCheckInterval);
      }
    }
    // 检查是否开始新一轮 CMS 回收周期
    if (_collector->shouldConcurrentCollect()) {
      return;
    }
  }
}
```

注：

* _CMSWaitDuration_：指的是 CMS 线程用于等待 Young GC 的时间，默认值是2000ms；
* _CMSIncrementalMode_ ：标识 CMS 是否使用 增量垃圾回收模式，默认为false；
* _wait_on_cms_lock_for_scavenge_：这个函数可以让线程 sleep 一段时间，除非有同步 GC 或者 Full GC 发生。

## 检查的过程

是否需要开始新一轮的 CMS 垃圾回收，主要实现在 _shouldConcurrentCollect();_

### 主动触发 GC

如果应用主动请求 GC（_System.gc()_），则直接触发 （当然要设置 _ExplicitGCInvokesConcurrent_ 参数才会使用 CMS 来完成 Full GC），判断的代码如下：

**ConcurrentMarkSweepGeneration.cpp**
```
if (_full_gc_requested) {
    if (Verbose && PrintGCDetails) {
        gclog_or_tty->print_cr("CMSCollector: collect because of explicit "
                                       " gc request (or gc_locker)");
    }
    return true;
}
```

### 被动触发检查

#### 未设置 _-XX:+UseCMSInitiatingOccupancyOnly_

* 如果统计开启了，且 CMS 完成时间小于 CMS 中各代剩余空间被填满的时间，则触发一次 CMS；
* 统计不可用，（第一次没有统计信息)，年老代大于__bootstrap_occupancy_，则触发 CMS ；

**ConcurrentMarkSweepGeneration.cpp**
```
//CMSBootstrapOccupancy 是一个可配置的参数，其值介于 0 到 100 之间。
_bootstrap_occupancy = ((double) CMSBootstrapOccupancy) / (double) 100;

if (!UseCMSInitiatingOccupancyOnly) {
    if (stats().valid()) {
        if (stats().time_until_cms_start() == 0.0) {
            return true;
        }
    } else {
        if (_cmsGen->occupancy() >= _bootstrap_occupancy) {
            if (Verbose && PrintGCDetails) {
                gclog_or_tty->print_cr(
                        " CMSCollector: collect for bootstrapping statistics:"
                                " occupancy = %f, boot occupancy = %f", _cmsGen->occupancy(),
                        _bootstrap_occupancy);
            }
            return true;
        }
    }
}
```

#### 设置了 _-XX:+UseCMSInitiatingOccupancyOnly_

1.  首先判断老年代内存使用决定是否回收，核心代码在 _should_concurrent_collect_ ，为 true 则触发 CMS；
2.  根据增量模式收集是否失败决定是否回收，_incremental_collection_will_fail_，为 true 则触发 CMS，返回为 true 的主要条件有两个：
    1.  老年代的可用内存大小 < Eden 区的内存使用量 + From 区的内存使用量
    2.  老年代的可用内存大小 < YGC 时晋升到老年代对象大小的平均值
3.  根据元空间的内存使用决定是否回收， _MetaspaceGC::should_concurrent_collect()_，true 则触发 CMS ；
4.  最后根据触发间隔(_CMSTriggerInterval_)，其值默认为-1，所以一般不走这个逻辑);

**ConcurrentMarkThread.cpp**
```
// 根据内存使用是否满足回收的条件进行判定
if (_cmsGen->should_concurrent_collect()) {
    if (Verbose && PrintGCDetails) {
        gclog_or_tty->print_cr("CMS old gen initiated");
    }
    return true;
}

// 根据CMS是否存在回收失败的风险进行判定
GenCollectedHeap *gch = GenCollectedHeap::heap();
assert(gch->collector_policy()->is_two_generation_policy(),
       "You may want to check the correctness of the following");
if (gch->incremental_collection_will_fail(true /* consult_young */)) {
    if (Verbose && PrintGCDetails) {
        gclog_or_tty->print("CMSCollector: collect because incremental collection will fail ");
    }
    return true;
}

// 根据元空间剩余空间大小判定
if (MetaspaceGC::should_concurrent_collect()) {
    if (Verbose && PrintGCDetails) {
        gclog_or_tty->print("CMSCollector: collect for metadata allocation ");
    }
    return true;
}

// 根据 CMS 触发的周期判定：
// a. 如果配置为0则返回为 true
// b. 如果自cms触发到现在为止的时间大于触发周期，返回 true
if (CMSTriggerInterval >= 0) {
    if (CMSTriggerInterval == 0) {
        // Trigger always
        return true;
    }

    if (stats().cms_time_since_begin() >= (CMSTriggerInterval / ((double) MILLIUNITS))) {
        if (Verbose && PrintGCDetails) {
            if (stats().valid()) {
                gclog_or_tty->print_cr(
                        "CMSCollector: collect because of trigger interval (time since last begin %3.7f secs)",
                        stats().cms_time_since_begin());
            } else {
                gclog_or_tty->print_cr("CMSCollector: collect because of trigger interval (first collection)");
            }
        }
        return true;
    }
}
```

---

_should_concurrent_collect_ 被用于判断是否应触发 CMS 回收老年代，核心判断如下：

* 判断 _occupancy()_ 是否大于 _init_occupancy()（通过-XX:CMSInitiatingOccupancyFraction=n 初始化）_，大于则触发；
* 如果设置了 _-XX:+UseCMSInitiatingOccupancyOnly_，直接返回，不再继续后面逻辑；
* 如果没有设置 _-XX:+UseCMSInitiatingOccupancyOnly_，则根据空闲链表（记录了还可以分配的内存）的使用，返回对应的 bool 值，这里就不具体展开了。

其中 i_nit_occupancy()_ 可以获取 变量 _initiating\_occupancy_ 的值，后者是通过 JVM 的启动参数 _CMSInitiatingOccupancyFraction_ 初始化的。

**ConcurrentMarkSweepGeneration.cpp**
```
bool ConcurrentMarkSweepGeneration::should_concurrent_collect() const {
    assert_lock_strong(freelistLock());
    if (occupancy() > initiating_occupancy()) {
        if (PrintGCDetails && Verbose) {
            gclog_or_tty->print(" %s: collect because of occupancy %f / %f  ",
                                short_name(), occupancy(), initiating_occupancy());
        }
        return true;
    }
    if (UseCMSInitiatingOccupancyOnly) {
        return false;
    }
    if (expansion_cause() == CMSExpansionCause::_satisfy_allocation) {
        if (PrintGCDetails && Verbose) {
            gclog_or_tty->print(" %s: collect because expanded for allocation ",
                                short_name());
        }
        return true;
    }
    if (_cmsSpace->should_concurrent_collect()) {
        if (PrintGCDetails && Verbose) {
            gclog_or_tty->print(" %s: collect because cmsSpace says so ",
                                short_name());
        }
        return true;
    }
    return false;
}
```

在 _CompactibleFreeListSpace.cpp_ 中，实现了_\_cmsSpace_→_should\_concurrent\_collect()_：

**CompactibleFreeListSpace.cpp**
```
// Support for concurrent collection policy decisions.
bool CompactibleFreeListSpace::should_concurrent_collect() const {
  // In the future we might want to add in frgamentation stats --
  // including erosion of the "mountain" into this decision as well.
  return !adaptive_freelists() && linearAllocationWouldFail();
}
```

## 过程总结

是否进行CMS回收，归纳如下：

1.  如果应用主动请求 GC，直接触发；
2.  是否设置 _UseCMSInitiatingOccupancyOnly_；
    1.  没有设置 _UseCMSInitiatingOccupancyOnly_
        1.  统计开启（_stats.valid_），统计的cms完成时间小于cms剩余空间被填满的时间，则触发；
        2.  统计不可用，（第一次没有统计信息，!_stats.valid_)，年老代大于 __bootstrap_occupancy_，则触发；
    2.  设置 _UseCMSInitiatingOccupancyOnly_
        1.  根据指定年老代的判断逻辑 _should_concurrent_collect_，true 则触发；
        2.  根据增量模式收集是否失败，_incremental_collection_will_fail_，true 则触发；
        3.  根据元数据区的判断逻辑 _should_concurrent_collect_，true 则触发；
        4.  最后根据触发间隔(_CMSTriggerInterval_，默认为-1，所以一般不走这个逻辑);