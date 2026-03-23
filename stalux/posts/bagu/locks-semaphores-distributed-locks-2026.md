---
title: 第十五节：锁，信号量和分布式锁：如何控制同一时间只有两个线程运行
abbrlink: locks-semaphores-distributed-locks-2026
date: 2026-03-23T10:57:56+08:00
updated: 2026-03-23T10:57:56+08:00
tags:
  - 操作系统
  - 学习
  - 八股
categories:
  - 操作系统
desc: 第十五节：锁，信号量和分布式锁：如何控制同一时间只有两个线程运行
---

# 第十五节：锁，信号量和分布式锁：如何控制同一时间只有两个线程运行

锁是并发面试里的高频主题。

这一节不只看“锁”本身，还要把它和几个容易一起出现的概念串起来：

1. 锁是怎么实现的
2. 如何控制同一时间只有 2 个线程运行
3. `wait / notify`、`Monitor`、生产者消费者模型之间是什么关系
4. 死锁为什么会发生
5. 分布式锁为什么和单机锁不是一回事
6. 在 Go 和 Java 里，这些概念分别落到哪些工具上

这一节会尽量以 Go 为主线来讲，因为你特别提到了要结合 Go 的 `sync` 包；同时会把 Java 里的对应写法一起对照起来。

## 1. 先从竞争条件开始

锁出现的根本原因，是多个线程或协程会同时争抢同一份共享资源。

例如多个执行流同时去修改同一个变量、同一个内存地址、同一个库存值、同一个队列时，就可能出现**竞争条件**。

所谓竞争条件，可以简单理解成：

- 多个线程对同一个资源同时读写
- 最终结果依赖于具体的执行时序
- 执行顺序一变，结果就可能变

所以最后资源的值往往是**不可预测的**。

比如最经典的 `i++`：

```text
读取 i
计算 i + 1
写回 i
```

它通常不是一个不可分割的原子动作，而是好几个步骤。

如果两个线程都同时做 `i++`，就可能都先读到旧值，再分别写回，最后导致一次更新丢失。

这里也引出两个基础概念：

- **原子操作**：操作不可再分割，中间不会被别的线程插进来破坏一致性
- **临界区**：访问共享资源的那段代码

并发控制，本质上就是在想办法保护临界区。

## 2. 解决竞争条件的两类思路

解决并发冲突，通常有两大类方法：

- 避免共享
- 控制共享

### 2.1 避免共享

最理想的方式，是尽量让每个线程只操作自己的数据。

在 Java 里，这类思路常被说成 `ThreadLocal`。

意思是：

- 给每个线程单独一份变量副本
- 线程之间不共享
- 既然不共享，也就没有竞争

但它的限制也很明显：

- 不是所有业务都能拆成“每个线程一份”
- 一旦多个线程确实要操作同一份状态，还是得回到同步问题

在 Go 里没有 `ThreadLocal` 这种标准库级别的常用抽象，因为 Go 的推荐方向通常不是“给线程绑定局部变量”，而是：

- 尽量通过 `channel` 传递数据
- 尽量通过“不要共享内存来通信，而要通过通信来共享内存”的方式减少竞争

但如果共享真的不可避免，就还是要用锁、原子操作、信号量这些工具。

### 2.2 控制共享

控制共享的典型方案有：

- 互斥锁
- 读写锁
- CAS
- 自旋
- 条件变量
- 信号量
- Monitor
- 分布式锁

它们不是互相替代的关系，而是面向不同层次、不同场景的工具。

## 3. CAS 是什么，为什么它经常和锁一起出现

CAS 的全称是 `Compare And Swap`，也就是“比较并交换”。

它通常可以抽象成这样一个函数：

```text
cas(address, expectedValue, newValue)
```

它的语义是：

1. 先看 `address` 里的当前值是不是 `expectedValue`
2. 如果是，就把它更新成 `newValue`
3. 如果不是，就什么都不做，并返回失败

可以把它理解成一种 CPU 提供的原子指令能力。

### 3.1 CAS 为什么能解决竞争问题

还是看 `i++`。

如果两个线程同时对 `i` 做加一：

```text
线程 1：读取 i = 0
线程 2：读取 i = 0

线程 1：计算目标值 1
线程 2：计算目标值 1

线程 1：cas(&i, 0, 1) 成功
线程 2：cas(&i, 0, 1) 失败
```

因为线程 1 成功后，`i` 已经变成了 `1`，线程 2 再拿旧期望值 `0` 去比较，就失败了。

失败的线程通常会重试：

```text
while (!cas(&i, old, old + 1)) {
    // 重试
}
```

这就是很多无锁算法和乐观并发控制的基础。

### 3.2 Go 和 Java 里 CAS 对应什么

在 Java 里，CAS 常见于：

- `AtomicInteger`
- `AtomicLong`
- `AtomicReference`
- `Unsafe`
- `VarHandle`
- `AbstractQueuedSynchronizer`

像这样：

```java
AtomicInteger counter = new AtomicInteger(0);
counter.incrementAndGet();
```

底层本质还是 CAS。

在 Go 里，CAS 对应的是 `sync/atomic` 包。

例如：

```go
package main

import (
	"sync/atomic"
)

func inc(v *atomic.Int64) {
	for {
		old := v.Load()
		if v.CompareAndSwap(old, old+1) {
			return
		}
	}
}
```

这里和 Java `AtomicInteger` 的思路是完全对应的：

- 先读旧值
- 计算新值
- 尝试 CAS
- 失败就重试

### 3.3 CAS 的优缺点

优点：

- 不一定要把线程挂起
- 在低冲突场景下性能很好
- 很适合做计数器、状态位、无锁结构

缺点：

- 高冲突时会不断重试，浪费 CPU
- 只能很好地处理较简单的原子状态更新
- 代码通常比直接加锁更难写、更难证明正确

所以可以记一句很实用的话：

**锁更直观，CAS 更底层。**

## 4. Test-And-Set 和锁的关系

除了 CAS，操作系统课程里还经常会提到 `Test-And-Set`，也就是 `TAS`。

它的目标通常是：

- 测试某个锁变量当前是不是空闲
- 如果空闲，就立刻把它设置成占用状态

一个很粗糙的抽象写法可以写成：

```text
tas(&lock) {
    return cas(&lock, 0, 1)
}
```

也就是说，从思想上看，`tas` 和 `cas` 是非常接近的。

如果锁变量 `lock` 初始值是 `0`：

- `0` 表示未加锁
- `1` 表示已加锁

那么获取锁就可以写成：

```text
while (!cas(&lock, 0, 1)) {
    // 什么也不做
}
```

退出临界区时再把它改回 `0`。

这其实就是一个很经典的自旋锁雏形。

## 5. 锁是如何实现的

最直观的锁模型就是：

- 进入临界区前 `lock`
- 离开临界区后 `unlock`

伪代码如下：

```text
enter(lock)

// 临界区代码

leave(lock)
```

从编程模型上看，锁比直接手写 CAS 更简单，因为你不需要每次都自己处理重试和状态更新。

### 5.1 自旋锁

如果线程拿不到锁，不是立刻睡眠，而是原地一直循环尝试，这就是**自旋锁**。

伪代码：

```text
enter() {
    while (!cas(&lock, 0, 1)) {
    }
}

leave() {
    lock = 0
}
```

优点：

- 不会立刻发生线程阻塞和唤醒
- 如果持锁时间很短，切换成本可能比阻塞更高，自旋反而划算

缺点：

- 会一直占着 CPU
- 如果持锁时间较长，自旋代价很高

所以自旋锁本质上是在拿 CPU 时间换上下文切换开销。

### 5.2 Go 和 Java 里“锁”的对应

Go `sync` 包里最直接的锁有：

- `sync.Mutex`
- `sync.RWMutex`

最常见的是：

```go
package main

import "sync"

var (
	mu sync.Mutex
	n  int
)

func add() {
	mu.Lock()
	n++
	mu.Unlock()
}
```

如果担心忘记释放，通常会写成：

```go
func add() {
	mu.Lock()
	defer mu.Unlock()
	n++
}
```

Java 里最直观的对应有两个：

1. `synchronized`
2. `ReentrantLock`

例如：

```java
private final Object lock = new Object();
private int n = 0;

public void add() {
    synchronized (lock) {
        n++;
    }
}
```

或者：

```java
private final ReentrantLock lock = new ReentrantLock();
private int n = 0;

public void add() {
    lock.lock();
    try {
        n++;
    } finally {
        lock.unlock();
    }
}
```

可以把它们对照记：

- Go `sync.Mutex` 对应 Java 里的互斥访问能力
- Go 没有语言关键字级别的 `synchronized`
- Java 的 `ReentrantLock` 是显式锁
- Go 的 `sync.Mutex` 不是可重入锁，Java 的 `synchronized` 和 `ReentrantLock` 都是可重入的

这点很重要。

### 5.3 Go 的 `Mutex` 为什么要特别注意不可重入

Go 里的 `sync.Mutex` 不能被同一个 goroutine 重复加锁。

如果一个函数已经拿到了锁，又在同一调用链里再次 `Lock()`，就可能把自己卡死。

这和 Java 的 `synchronized`、`ReentrantLock` 不一样。

Java 的可重入锁允许：

- 同一个线程重复进入同一把锁保护的代码
- 内部会记录持有次数
- 对应次数的 `unlock` 后才真正释放

而 Go 标准库没有直接提供可重入锁，因为 Go 更鼓励：

- 缩小临界区
- 避免复杂锁嵌套
- 用更清晰的结构代替重入依赖

## 6. Monitor、wait、notify 到底是什么

很多人一提锁，就会顺便提到 `wait / notify`、阻塞队列、生产者消费者、`Monitor`。

这几个概念其实是连在一起的。

### 6.1 先理解 Monitor

可以把 `Monitor` 理解成一种更完整的并发控制模型，它通常不只是“一把锁”，还会包含：

- 互斥进入能力
- 等待队列
- 条件等待与唤醒机制

所以它不只是“谁能进临界区”，还包括：

- 进不去的人先去哪里等
- 条件满足后谁来唤醒
- 被唤醒后怎么重新竞争

在 Java 里，每个对象都可以和 `Monitor` 机制关联。

例如：

```java
synchronized (obj) {
    // 临界区
}
```

这里的 `obj` 就可以看成一把监视器锁关联对象。

### 6.2 wait 是做什么的

`wait` 的核心不是“睡一会儿”，而是：

- 当前线程释放持有的监视器
- 进入等待集合
- 等待别人发信号唤醒

所以它和纯自旋完全不同。

自旋是：

- 拿不到锁就一直空转

`wait` 是：

- 条件不满足就进入等待队列
- 不再持续占用 CPU

### 6.3 notify 是做什么的

`notify` 的作用是：

- 从等待集合里唤醒一个线程

`notifyAll` 则是：

- 唤醒所有等待线程

被唤醒的线程不是“立刻执行临界区”，而是先回到锁竞争流程，重新抢锁。

### 6.4 Java 和 Go 的对应关系

Java 里最典型的是：

```java
synchronized (lock) {
    while (!condition) {
        lock.wait();
    }
    // 临界区
}
```

以及：

```java
synchronized (lock) {
    updateState();
    lock.notifyAll();
}
```

Go 没有对象内建的 `wait / notify` 语法，但标准库里有直接对应的工具：`sync.Cond`。

例如：

```go
package main

import "sync"

var (
	mu   sync.Mutex
	cond = sync.NewCond(&mu)
	ready bool
)

func waitReady() {
	mu.Lock()
	for !ready {
		cond.Wait()
	}
	mu.Unlock()
}

func signalReady() {
	mu.Lock()
	ready = true
	cond.Broadcast()
	mu.Unlock()
}
```

可以这样对照：

- Java `wait()` 对应 Go `cond.Wait()`
- Java `notify()` 对应 Go `cond.Signal()`
- Java `notifyAll()` 对应 Go `cond.Broadcast()`

而且两边都要注意一件事：

- **条件等待通常要放在循环里判断，而不是只判断一次**

因为被唤醒不代表条件一定真的满足，也可能是别的线程先把条件又改掉了。

## 7. 生产者消费者模型为什么总和锁放在一起讲

生产者消费者模型是并发控制里最经典的例子之一。

它要解决的是：

- 生产者往缓冲区放数据
- 消费者从缓冲区取数据
- 缓冲区满了，生产者要等
- 缓冲区空了，消费者要等

这个问题天然同时需要：

- 互斥
- 等待
- 唤醒
- 容量计数

所以它非常适合用来理解 `Monitor`、条件变量、信号量。

### 7.1 Java 中常见的理解方式

如果用 Java 的 `synchronized + wait + notifyAll` 来写，本质就是：

- 用 `synchronized` 保护共享队列
- 队列满时，生产者 `wait`
- 队列空时，消费者 `wait`
- 成功放入或取出后，再 `notifyAll`

### 7.2 Go 中更常见的对应

Go 里有两种典型写法：

1. 直接用 `channel`
2. 用 `sync.Mutex + sync.Cond`

如果只是教学上理解生产者消费者，`channel` 很直观：

```go
package main

func producer(ch chan<- int, v int) {
	ch <- v
}

func consumer(ch <-chan int) int {
	return <-ch
}
```

有缓冲 `channel` 本身就带容量概念。

但如果你是在讲“锁、条件变量、信号量”的原理，那更接近 Java `Monitor` 写法的是 `sync.Cond`。

## 8. 信号量是什么，为什么它能控制只有两个线程运行

互斥锁的特点是：

- 同一时刻只允许 1 个线程进入临界区

但有些场景不是“只许一个”，而是：

- 同一时间最多允许 `N` 个线程进入

这就是**信号量**的典型用途。

你可以把信号量理解成一个“许可证计数器”。

- 有几个许可证，就允许几个线程进入
- 线程进入时先拿一个许可证
- 线程离开时再归还一个许可证

### 8.1 P/V 操作，也就是 down/up

操作系统教材里常把信号量的两个基本操作写成：

- `P` 或 `down`
- `V` 或 `up`

它们的语义可以记成：

- `down`：申请一个资源，如果没有资源就等待
- `up`：释放一个资源，并唤醒等待者

如果信号量初始值是：

- `1`，那它的效果很像互斥锁
- `2`，那就表示同一时间最多允许 2 个线程进入
- `N`，那就表示同一时间最多允许 `N` 个线程进入

### 8.2 如何控制同一时间只有两个线程运行

思路非常直接：

- 准备一个初始值为 `2` 的信号量
- 进入临界区前执行 `down`
- 离开临界区后执行 `up`

这样任意时刻最多只有两个执行流能拿到许可证。

### 8.3 Java 里怎么写

Java 标准库直接有 `Semaphore`：

```java
import java.util.concurrent.Semaphore;

Semaphore semaphore = new Semaphore(2);

public void work() {
    try {
        semaphore.acquire();
        // 临界区，最多同时 2 个线程进来
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
    } finally {
        semaphore.release();
    }
}
```

这里的 `acquire()` 就对应 `down`，`release()` 就对应 `up`。

### 8.4 Go 里怎么写

Go 标准库 `sync` 包里**没有内建的 `Semaphore` 类型**，这是一个很容易被问到的点。

但 Go 里通常有三种写法：

1. 用带缓冲 `channel`
2. 用 `golang.org/x/sync/semaphore`
3. 用 `sync.Cond` 和计数器自己实现

如果要贴近 Go 日常开发，最常见的是带缓冲 `channel`：

```go
package main

var sem = make(chan struct{}, 2)

func work() {
	sem <- struct{}{}
	defer func() { <-sem }()

	// 临界区，最多同时 2 个 goroutine 执行
}
```

这里可以这样理解：

- `sem` 容量是 `2`
- 往里塞一个空结构体，表示拿到一个名额
- 满了之后，再塞就会阻塞
- 离开时取出一个，表示归还名额

如果你硬要和 Java `Semaphore` 做一一对应，那这个 `channel` 就是在扮演“许可证池”的角色。

从原理上说，这就是信号量思想。

## 9. 信号量和互斥锁的区别

这两个概念很容易混。

可以这样区分：

- **互斥锁**：同一时间只允许 1 个线程进入
- **信号量**：同一时间允许最多 `N` 个线程进入

所以：

- 互斥锁是特殊情况的信号量
- 初始值为 `1` 的信号量，效果上很接近互斥

但实际工程里两者的抽象目标还是不同：

- 互斥锁更强调保护临界区
- 信号量更强调控制并发资源数量

## 10. 读写锁属于什么角色

有时候共享资源不是“谁都在写”，而是：

- 读很多
- 写很少

这时候普通互斥锁会显得太保守，因为它让读和读之间也互斥了。

读写锁的思路是：

- 读和读可以并发
- 写和任何操作互斥

Go 里对应的是 `sync.RWMutex`：

```go
package main

import "sync"

var rw sync.RWMutex
var data int

func read() int {
	rw.RLock()
	defer rw.RUnlock()
	return data
}

func write(v int) {
	rw.Lock()
	defer rw.Unlock()
	data = v
}
```

Java 里常见对应是 `ReentrantReadWriteLock`。

所以也可以顺手记住：

- Go `sync.Mutex` <-> Java `synchronized` / `ReentrantLock`
- Go `sync.RWMutex` <-> Java `ReentrantReadWriteLock`

## 11. 死锁是怎么发生的

死锁本质上可以理解成一种**环状依赖**。

最经典的情况是：

- 线程 1 拿到了锁 A，接着等锁 B
- 线程 2 拿到了锁 B，接着等锁 A

双方都在等对方放手，于是谁也走不下去。

伪代码：

```text
线程 1：
先拿 lock1
再拿 lock2

线程 2：
先拿 lock2
再拿 lock1
```

如果时序刚好错开，就死锁了。

### 11.1 Go 和 Java 里都会遇到同样的问题

不管你是 Go 的 `sync.Mutex`，还是 Java 的 `synchronized` / `ReentrantLock`，只要存在多把锁交叉获取，就可能死锁。

### 11.2 常见预防方式

最常见、也最实用的办法是：

- 统一加锁顺序

比如规定：

- 所有人都必须先拿 `lock1`
- 再拿 `lock2`

这样就不会出现环状等待。

其他办法还包括：

- 缩小临界区
- 避免嵌套锁
- 尝试锁超时
- 用更高层的并发模型重构

## 12. 分布式锁为什么单机锁不够用

单机多线程时，原子操作通常由 CPU 指令保证，比如：

- CAS
- TAS

但到了分布式场景，多个线程可能已经不在同一台机器上了，而是：

- 多个进程
- 多台机器
- 多个服务实例

这时候你没法再靠本地内存地址上的 CAS 来解决问题。

例如有 100 个服务实例都去做“扣减库存”：

1. 从 Redis 读库存
2. 计算库存减一
3. 写回 Redis

如果没有额外同步，这 3 步就不是整体原子的，还是会出现竞争。

所以分布式锁的核心问题其实是：

**在分布式环境下，原子性由谁提供？**

答案通常是：

- Redis 的原子命令，比如 `SET NX`
- ZooKeeper 的节点创建与顺序节点机制
- 数据库的唯一约束、事务、悲观锁、乐观锁

也就是说，分布式锁不是“把本地锁搬到网络上”，而是借助一个**外部协调者**来提供跨进程的原子能力。

## 13. Redis 分布式锁怎么理解

最常见的思路是基于 Redis：

```text
SET lock_key request_id NX PX 30000
```

它的意思是：

- 只有当 `lock_key` 不存在时才设置成功
- 设置成功就表示拿到锁
- 同时带过期时间，避免持锁方挂掉后永远不释放

这里的关键点有 3 个：

1. **加锁必须是原子的**
2. **锁要带过期时间**
3. **释放锁时要校验锁的持有者**

为什么释放锁时要校验？

因为可能发生这种情况：

- 线程 A 拿到锁，但执行太久，锁过期了
- 线程 B 又拿到了同一把锁
- 线程 A 这时恢复执行，如果它直接 `DEL lock_key`，就把线程 B 的锁删掉了

所以正确思路是：

- 锁值里要存一个唯一 `request_id`
- 删除时先比对是不是自己的锁

通常要用 Lua 脚本保证“比较并删除”也是原子操作。

## 14. Go 和 Java 里如何看待分布式锁

这一层其实已经不再是语言级锁了，而是中间件能力。

Go 和 Java 的区别主要体现在客户端库，不体现在原理。

例如：

- Java 里常见 `Redisson`
- Go 里常见 Redis 客户端配合 Lua、或者用 etcd / ZooKeeper / Redis 实现

它们本质上都是：

- 向一个共享协调节点申请锁
- 拿到锁后执行业务
- 结束后释放锁

所以可以记一句话：

**单机锁解决的是同一进程内的并发；分布式锁解决的是跨进程、跨机器的并发。**

## 15. 从 Go `sync` 包回看这一节

你特别要求结合 Go 的 `sync` 包，那这一节最后可以统一收束一下。

Go 里和本节最相关的工具主要有：

- `sync.Mutex`：互斥锁，保护临界区
- `sync.RWMutex`：读写锁，适合读多写少
- `sync.Cond`：条件等待和唤醒，对应 Java 的 `wait / notify`
- `sync.Once`：保证只执行一次，本质也是并发控制
- `sync.WaitGroup`：等待一组 goroutine 完成，它不是锁，但经常和锁一起出现

而底层原子能力更多在：

- `sync/atomic`

所以可以把这几个层次这样理解：

1. `sync/atomic`
   更底层，提供 CAS、原子读写、原子加减
2. `sync.Mutex` / `sync.RWMutex`
   更高层，提供更直观的临界区保护
3. `sync.Cond`
   在锁的基础上继续提供等待和通知语义
4. `channel`
   Go 风格的更高层并发通信工具，很多场景可以减少显式加锁

## 16. 一道面试题串起来看

如果面试官问：

**如何控制同一时间只有 2 个线程运行？**

一个比较完整的回答是：

1. 如果只是限制并发数，本质上该用信号量，而不是普通互斥锁
2. Java 可以直接用 `Semaphore(2)`
3. Go 标准库 `sync` 没有直接提供 `Semaphore` 类型，但可以用容量为 `2` 的带缓冲 `channel` 实现
4. 如果一定从操作系统原理讲，信号量本质是一个带等待队列的计数器，`down` 申请，`up` 释放
5. 如果进一步追问底层，可以继续讲 CAS、自旋、阻塞、等待队列

## 17. 这一节先记住的点

- 竞争条件的本质是多个线程对同一共享资源的访问顺序不可控。
- `i++` 通常不是原子操作，所以会有丢失更新问题。
- CAS 是“比较并交换”，是很多原子类和锁实现的重要基础。
- 锁更直观，CAS 更底层；低冲突时 CAS 很有价值，高冲突时会浪费 CPU。
- 自旋锁是在循环里不断尝试获取锁，不主动阻塞，优点是少切换，缺点是耗 CPU。
- Java 的 `synchronized` 和 `ReentrantLock` 是可重入的，Go 的 `sync.Mutex` 不是可重入锁。
- `wait / notify` 背后是 `Monitor` 思想；Go 中更接近的工具是 `sync.Cond`。
- 信号量适合控制“同时最多允许 N 个线程进入”，控制 2 个线程并发运行时最适合用信号量。
- Java 有现成的 `Semaphore`；Go 可以用带缓冲 `channel` 或额外信号量库实现。
- 死锁可以理解成环状依赖，最常见的预防方式是统一加锁顺序。
- 分布式锁不是本地锁的延伸，而是依赖 Redis、ZooKeeper、数据库等外部系统提供跨进程原子性。
