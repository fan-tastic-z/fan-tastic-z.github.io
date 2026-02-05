---
title: rust 并发基础
date: 2026-02-03 10:12:58
tags:
published: false
---

通过阅读一些优秀的代码慢慢的学习rust并发相关的知识。

## Atomic 

使用Rust Atomic的时候关于"内存顺序" 概念，通过一些例子进行更好的理解。

### 内存顺序的五个规则

- Relaxed: 只保证该原子操作本身原子，不提供跨线程的先后关系；适合无跨线程依赖的计数。
- Release: 在写入时，保证当前线程在此之前的普通写入，对其他线程在后续 Acquire 读到该写入时可见。
- Acquire：在读取时，保证当它读到某个 Release 写入后，随后在本线程的普通读写不可“越过”这次读取。
- AcqRel：读改写操作同时具备 Acquire 与 Release 语义（如 fetch_add 返回旧值且发布新值）。
- SeqCst：最强，除 Acquire/Release 语义外，还在所有 SeqCst 原子操作之间建立单一全序（对调试友好，性能最差）。


```rust
// Message passing with relaxed order can read stale data even after seeing the flag.
#[test]
#[should_panic]
fn message_passing_relaxed_can_fail() {
    loom::model(|| {
        let data = Arc::new(AtomicUsize::new(0));
        let flag = Arc::new(AtomicBool::new(false));

        let data_w = data.clone();
        let flag_w = flag.clone();
        let writer = thread::spawn(move || {
            data_w.store(1, Ordering::Relaxed);
            flag_w.store(true, Ordering::Relaxed);
        });

        let data_r = data.clone();
        let flag_r = flag.clone();
        let reader = thread::spawn(move || {
            if flag_r.load(Ordering::Relaxed) {
                // With Relaxed, loom can schedule so that this still reads 0.
                assert_eq!(data_r.load(Ordering::Relaxed), 1);
            }
        });

        writer.join().unwrap();
        reader.join().unwrap();
    });
}

// Message passing with Release/Acquire prevents the stale read once the flag is observed.
#[test]
fn message_passing_release_acquire_is_safe() {
    loom::model(|| {
        let data = Arc::new(AtomicUsize::new(0));
        let flag = Arc::new(AtomicBool::new(false));

        let data_w = data.clone();
        let flag_w = flag.clone();
        let writer = thread::spawn(move || {
            data_w.store(1, Ordering::Relaxed);
            flag_w.store(true, Ordering::Release);
        });

        let data_r = data.clone();
        let flag_r = flag.clone();
        let reader = thread::spawn(move || {
            if flag_r.load(Ordering::Acquire) {
                // Acquire sees the Release, so the data write is visible.
                assert_eq!(data_r.load(Ordering::Relaxed), 1);
            }
        });

        writer.join().unwrap();
        reader.join().unwrap();
    });
}
```


前面两个代码例子相对来说比较好理解，我们看看下面这个例子：

`store_buffering_relaxed_allows_both_zero`（标记 should_panic）

- 两个原子布尔 x/y 初始 false。T1：x.store(true, Relaxed) 再 y.load(Relaxed)；T2：y.store(true, Relaxed) 再 x.load(Relaxed)。
- Relaxed 允许 CPU/编译器把 store 留在写缓冲，load 直接从对方的缓存或初始值读，彼此的写暂时不可见。loom 会找到一条合法执行：T1 的 load 看到 y=false，T2 的 load 看到 `x=false`，两边都“错过”了对方的 store。
- 断言 assert!(r1 || r2); 要求至少有一边看到 true；但在这条合法执行里 `r1=false && r2=false`，断言失败触发 panic。因为标记了 `#[should_panic]`，测试反而“通过”，证明 Relaxed允许这种违背直觉的结果

`store_buffering_seqcst_disallows_both_zero`

- 同样的操作，但所有原子操作用 SeqCst。除了 Acquire/Release 语义外，SeqCst 会把所有 SeqCst 原子放入一个全局单序。在线性化顺序下，不可能让两个 load 都先于两个 store：至少有一个 store 在线性序列中出现在另一个线程的 load 之前，因此至少有一个 load 必须看到 true。
- loom 穷举时找不到 r1=false && r2=false 的执行，断言 assert!(r1 || r2); 始终成立，测试通过，从而展示 SeqCst 排除了 Relaxed 可见的“双方都没看见对方的写”结果。

```rust
// Store buffering: both threads exchange a bit; under Relaxed both loads can legally see 0.
#[test]
#[should_panic]
fn store_buffering_relaxed_allows_both_zero() {
    loom::model(|| {
        let x = Arc::new(AtomicBool::new(false));
        let y = Arc::new(AtomicBool::new(false));

        let xa = x.clone();
        let yb = y.clone();
        let t1 = thread::spawn(move || {
            xa.store(true, Ordering::Relaxed);
            let r1 = yb.load(Ordering::Relaxed);
            // Under Relaxed, loom can find r1 == false while the other thread also misses this store.
            r1
        });

        let xb = x.clone();
        let ya = y.clone();
        let t2 = thread::spawn(move || {
            ya.store(true, Ordering::Relaxed);
            let r2 = xb.load(Ordering::Relaxed);
            r2
        });

        let r1 = t1.join().unwrap();
        let r2 = t2.join().unwrap();

        // Loom should find an execution where both reads see false (store buffering).
        assert!(r1 || r2);
    });
}

// With SeqCst, the store-buffering outcome where both loads see false is disallowed.
#[test]
fn store_buffering_seqcst_disallows_both_zero() {
    loom::model(|| {
        let x = Arc::new(AtomicBool::new(false));
        let y = Arc::new(AtomicBool::new(false));

        let xa = x.clone();
        let yb = y.clone();
        let t1 = thread::spawn(move || {
            xa.store(true, Ordering::SeqCst);
            let r1 = yb.load(Ordering::SeqCst);
            r1
        });

        let xb = x.clone();
        let ya = y.clone();
        let t2 = thread::spawn(move || {
            ya.store(true, Ordering::SeqCst);
            let r2 = xb.load(Ordering::SeqCst);
            r2
        });

        let r1 = t1.join().unwrap();
        let r2 = t2.join().unwrap();

        assert!(r1 || r2);
    });
}
```

### Atomic 常用方法

`saturating_add/sub`: 整数加减到极值后，不溢出

```rust
fn main() {
     let a: u32 = u32::MAX;
     let b = a.saturating_add(10);
     println!("{}", b); // stays at u32::MAX
 }
```

`compare_exchange_weak`：CAS，可能“伪失败”，因此要放在循环里；成功时写入新值并返回 `Ok(old)`，失败时返回 `Err(actual)`

```rust
fn cas_inc(counter: &AtomicU32) -> u32 {
     let mut cur = counter.load(Ordering::Acquire);
     loop {
         let next = cur.saturating_add(1);
         match counter.compare_exchange_weak(cur, next, Ordering::Release, Ordering::Relaxed) {
             Ok(_) => return next,
             Err(actual) => cur = actual, // refresh expected and retry
         }
     }
 }
 ```

`fetch_add / fetch_sub`：原子加减，返回旧值

```rust
use std::sync::atomic::{AtomicI64, Ordering};
let v = AtomicI64::new(0);
let old = v.fetch_add(5, Ordering::AcqRel);
```

`fetch_update`：带闭包的更新，内部自动循环，失败会重试；适合条件更新。

```rust
use std::sync::atomic::{AtomicU32, Ordering};
let v = AtomicU32::new(2);
let res = v.fetch_update(Ordering::AcqRel, Ordering::Acquire, |cur| {
    (cur % 2 == 0).then_some(cur + 1)
});
```

`fetch_update(success_order, failure_order, f)` 的前两个参数是内存序，通常取法是“成功路径用 `AcqRel`（或 `Release/Acquire`），失败路径用 `Acquire` 或 `Relaxed`"，保证：更新成功时对后续读可见，失败重试时读到的值至少是 Acquire 语义。
简要说明：
- `success_order`：闭包返回 Some，CAS 成功写入新值时使用的内存序。常用：
  - `Ordering::AcqRel`：既释放又获取，通用安全。
  - `Ordering::Release`：如果你只需要发布写入、对当前线程不要求获得前序写的可见性。
  - `Ordering::SeqCst`：全局顺序最强，但性能差。
- `faile_order`：CAS 失败时用于读取当前值（返回 Err 时的 load 语义）。常用：
  - `Orring::Acquire`：确保看到最新写入（匹配成功时的 Release/AcqRel）。
  - `Ordeng::Relaxed`：如果失败路径只用于重试、不依赖其它数据可见性，可以放宽。

 实例 1（通用安全版，常用）：

```rust
let res = v.fetch_update(Ordering::AcqRel, Ordering::Acquire, |cur| {
      (cur % 2 == 0).then_some(cur + 1)
  });
```

实例 2（读少、只发布写入的场景，可以略放宽）：
```rust
let res = v.fetch_update(Ordering::Release, Ordering::Relaxed, |cur| {
     (cur % 2 == 0).then_some(cur + 1)
 });
```

记忆法：

- 成功：要写新值 → 至少 Release；若后续还要读到前序写 → 用 AcqRel。
- 失败：只读当前值 → Acquire 保守安全；不依赖可见性 → Relaxed。

建议：在不需要极致性能时默认 AcqRel/Acquire，这是 “安全默认”。需要放宽时，再基于数据依赖和性能权衡做调整。

### 实现一个精简版的 WaitGroup

```rust
use std::sync::{
    Arc,
    atomic::{AtomicUsize, Ordering},
};

use tokio::sync::Notify;

#[tokio::main]
async fn main() {
    let wg = WaitGroup::new();
    for i in 0..5 {
        let worker_wg = wg.clone_worker();
        tokio::spawn(async move {
            // Simulate some work with a sleep
            println!("Worker {} starting", i);
            tokio::time::sleep(tokio::time::Duration::from_millis(100 * i)).await;
            println!("Worker {} done", i);
            drop(worker_wg); // Indicate this worker is done
        });
    }
    wg.wait().await; // Wait for all workers to finish
    println!("All workers completed");
}

#[derive(Clone)]
pub struct WaitGroup {
    inner: Arc<Inner>,
}

struct Inner {
    count: AtomicUsize,
    notify: Notify,
}

impl WaitGroup {
    pub fn new() -> Self {
        Self {
            inner: Arc::new(Inner {
                count: AtomicUsize::new(1),
                notify: Notify::new(),
            }),
        }
    }

    pub fn clone_worker(&self) -> Self {
        self.inner
            .count
            .fetch_add(1, std::sync::atomic::Ordering::SeqCst);
        Self {
            inner: Arc::clone(&self.inner),
        }
    }

    pub async fn wait(self) {
        self.done();
        loop {
            if self.inner.count.load(Ordering::SeqCst) == 0 {
                break;
            }
            self.inner.notify.notified().await;
        }
    }

    fn done(&self) {
        if self.inner.count.fetch_sub(1, Ordering::SeqCst) == 1 {
            self.inner.notify.notify_waiters();
        }
    }
}

impl Drop for WaitGroup {
    fn drop(&mut self) {
        self.done();
    }
}
```

## 相关链接

- https://github.com/fast/mea
