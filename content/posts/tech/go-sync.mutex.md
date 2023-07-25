---
title: "Go sync.mutex源码解析" #标题
date: 2023-02-02T17:13:40+08:00 #创建时间
lastmod: 2023-02-02T17:13:40+08:00 #更新时间
author: ["Younger"] #作者
categories:
- 分类1
- 分类2
tags:
- golang
- 源码
description: "" #描述
weight: # 输入1可以顶置文章，用来给文章展示排序，不填就默认按时间排序
slug: ""
draft: false # 是否为草稿
comments: true #是否展示评论
showToc: true # 显示目录
TocOpen: true # 自动展开目录
hidemeta: false # 是否隐藏文章的元信息，如发布日期、作者等
disableShare: true # 底部不显示分享栏
showbreadcrumbs: true #顶部显示当前路径
cover:
image: "" #图片路径：posts/tech/文章1/picture.png
caption: "" #图片底部描述
alt: ""
relative: false
---
![Lock](https://youngergo.cn/images/tech/go-sync.mutex/go-sync-mutext.jpg)

### 开端
今天学习下go里面的sync.mutex的实现以及相关扩展知识。

### 锁的介绍
首先，计算机中的锁是为了控制并发情况下，对同一资源的并发访问。锁呢，有利有弊。好的点在于，我们可以控制并发访问的顺序逻辑。避免程序因为资源竞争，而出现一些预期外的情况。
不好的点在于，加锁意味着并发度的下降，效率的下降。所以我们在使用锁来完成业务需求的时候，也要考虑锁竞争对业务带来的影响。根据业务情况确定是否使用及使用的方式。

### 数据结构
```
// A Mutex is a mutual exclusion lock.
// The zero Val for a Mutex is an unlocked mutex.
//
// A Mutex must not be copied after first use.
type Mutex struct {
	state int32
	sema  uint32
}

// A Locker represents an object that can be locked and unlocked.
type Locker interface {
	Lock()
	Unlock()
}

const (
	mutexLocked = 1 << iota // mutex is locked
	mutexWoken
	mutexStarving
	mutexWaiterShift = iota

	// Mutex fairness.
	//
	// Mutex can be in 2 modes of operations: normal and starvation.
	// In normal mode waiters are queued in FIFO order, but a woken up waiter
	// does not own the mutex and competes with new arriving goroutines over
	// the ownership. New arriving goroutines have an advantage -- they are
	// already running on CPU and there can be lots of them, so a woken up
	// waiter has good chances of losing. In such case it is queued at front
	// of the wait queue. If a waiter fails to acquire the mutex for more than 1ms,
	// it switches mutex to the starvation mode.
	//
	// In starvation mode ownership of the mutex is directly handed off from
	// the unlocking goroutine to the waiter at the front of the queue.
	// New arriving goroutines don't try to acquire the mutex even if it appears
	// to be unlocked, and don't try to spin. Instead they queue themselves at
	// the tail of the wait queue.
	//
	// If a waiter receives ownership of the mutex and sees that either
	// (1) it is the last waiter in the queue, or (2) it waited for less than 1 ms,
	// it switches mutex back to normal operation mode.
	//
	// Normal mode has considerably better performance as a goroutine can acquire
	// a mutex several times in a row even if there are blocked waiters.
	// Starvation mode is important to prevent pathological cases of tail latency.
	starvationThresholdNs = 1e6
)
```

上面代码就是go1.18中sync.mutex的定义。可以看到Mutex结构体中有state和sema两个字段，
- state int32类型，代表的是锁的状态.
- sema uint32类型，代表[信号量](https://zh.wikipedia.org/wiki/%E4%BF%A1%E5%8F%B7%E9%87%8F)。他主要用于唤醒阻塞在互斥锁上的其他协程。

#### 锁的状态
![m.state](https://youngergo.cn/images/tech/go-sync.mutex/mutexStruct.png)

图片来自Draveness大神，state有以下几种状态：

- mutexLocked。锁状态，占1bit，0-可以获取锁，1-锁定状态，阻塞等待
- mutexWoken。唤醒状态。占1bit，代表一个过程阶段，0-没有协程唤醒 1-有协程被唤醒，申请锁定过程中。
- mutexStarving。饥饿状态。占1bit，当协程超过1ms还没有获取锁时，锁就会处于饥饿状态。
- mutexWaiterShift/waitersCount。等待信号量状态。占29bit，当有协程释放锁时，需要根据此状态，决定是否释放信号量。用于通知其他协程获取此锁。

### 实现原理
sync.Mutex的实现代码只有200多行，但是里面锁的切换控制还是比较复杂，下面我们逐步来分析

#### 加锁过程
```go
// Lock locks m.
// If the lock is already in use, the calling goroutine
// blocks until the mutex is available.
func (m *Mutex) Lock() {
	// Fast path: grab unlocked mutex.
	if atomic.CompareAndSwapInt32(&m.state, 0, mutexLocked) {
		if race.Enabled {
			race.Acquire(unsafe.Pointer(m))
		}
		return
	}
	// Slow path (outlined so that the fast path can be inlined)
	m.lockSlow()
}
```

Lock函数使用CompareAndSwapInt32判断m.state是不是0，如果是的话state设为lock状态，成功获取到锁。
失败则调用lockSlow()函数，这个函数是实现状态控制的主要逻辑。

```go
func (m *Mutex) lockSlow() {
	var waitStartTime int64
	starving := false
	awoke := false
	iter := 0
	old := m.state
	for {
		// Don't spin in starvation mode, ownership is handed off to waiters
		// so we won't be able to acquire the mutex anyway.
		if old&(mutexLocked|mutexStarving) == mutexLocked && runtime_canSpin(iter) {
			// Active spinning makes sense.
			// Try to set mutexWoken flag to inform Unlock
			// to not wake other blocked goroutines.
			if !awoke && old&mutexWoken == 0 && old>>mutexWaiterShift != 0 &&
				atomic.CompareAndSwapInt32(&m.state, old, old|mutexWoken) {
				awoke = true
			}
			runtime_doSpin()
			iter++
			old = m.state
			continue
		}
		new := old
		// Don't try to acquire starving mutex, new arriving goroutines must queue.
		if old&mutexStarving == 0 {
			new |= mutexLocked
		}
		if old&(mutexLocked|mutexStarving) != 0 {
			new += 1 << mutexWaiterShift
		}
		// The current goroutine switches mutex to starvation mode.
		// But if the mutex is currently unlocked, don't do the switch.
		// Unlock expects that starving mutex has waiters, which will not
		// be true in this case.
		if starving && old&mutexLocked != 0 {
			new |= mutexStarving
		}
		if awoke {
			// The goroutine has been woken from sleep,
			// so we need to reset the flag in either case.
			if new&mutexWoken == 0 {
				throw("sync: inconsistent mutex state")
			}
			new &^= mutexWoken
		}
		if atomic.CompareAndSwapInt32(&m.state, old, new) {
			if old&(mutexLocked|mutexStarving) == 0 {
				break // locked the mutex with CAS
			}
			// If we were already waiting before, queue at the front of the queue.
			queueLifo := waitStartTime != 0
			if waitStartTime == 0 {
				waitStartTime = runtime_nanotime()
			}
			runtime_SemacquireMutex(&m.sema, queueLifo, 1)
			starving = starving || runtime_nanotime()-waitStartTime > starvationThresholdNs
			old = m.state
			if old&mutexStarving != 0 {
				// If this goroutine was woken and mutex is in starvation mode,
				// ownership was handed off to us but mutex is in somewhat
				// inconsistent state: mutexLocked is not set and we are still
				// accounted as waiter. Fix that.
				if old&(mutexLocked|mutexWoken) != 0 || old>>mutexWaiterShift == 0 {
					throw("sync: inconsistent mutex state")
				}
				delta := int32(mutexLocked - 1<<mutexWaiterShift)
				if !starving || old>>mutexWaiterShift == 1 {
					// Exit starvation mode.
					// Critical to do it here and consider wait time.
					// Starvation mode is so inefficient, that two goroutines
					// can go lock-step infinitely once they switch mutex
					// to starvation mode.
					delta -= mutexStarving
				}
				atomic.AddInt32(&m.state, delta)
				break
			}
			awoke = true
			iter = 0
		} else {
			old = m.state
		}
	}

	if race.Enabled {
		race.Acquire(unsafe.Pointer(m))
	}
}
```
slowLock函数依靠CAS+信号量+自旋来实现。下面我们对照代码，逐行分析下逻辑： 

- 1.在调用slowLock之前，已经判断过state是否可以获取锁。进入slowLock后会阻塞当前G，尝试获取锁。 首先判断是否满足条件：
【锁处于非饥饿状态, locked状态，并且可以自旋】，满足即开始自旋，在自旋的过程中尝试将锁的状态设置为唤醒，尽量让当前G获取的锁。

- 2.当在自旋的过程中发现可以获取锁时，进入下面逻辑。用old初始化一个临时的new state。判断锁如果不处于饥饿状态，new state加上locked状态。
(饥饿状态下的锁，G是需要排队才可以获取的，源码注释也有)。接着，判断old状态是locked或者饥饿时，将waiterShift加8，代表等待的G加1。
接着判断如果此G已经处于饥饿状态，并且old已经处于locked状态。new state就增加饥饿状态。(文中的翻译说明了原因，当前的goroutine将mutex设为饥饿状态，
但是如果mutex已经解锁的话，就不要进行设定了。因为Unlock需要处于饥饿状态的mutex有等待者)。 判断当前G是否是被唤醒的，是的话将new state设置成非唤醒。

- 3.接下来要注意，这里再次将m.state和old进行了比较。主要是判断有其他的goroutine改变mutex的状态。如果有的话new state就作废，完成本次逻辑，继续循环自旋尝试获取锁。
如果没有的话，当前无锁+不饥饿就可以获取锁成功break。否则加锁失败，G需要进入队列，如果G第一次等待就放在队尾，否则就是被唤醒再次获取锁失败。就会被放在队首。
判断当前G是否该进入饥饿状态的标准是G等待mutex的时间是否大于1ms。重新获取mutex的状态，如果处于饥饿状态获取锁成功。此时需要考虑解除锁的饥饿状态。
满足1.当前G等待时间小于1ms 2.等待队列为空 两个条件其一即可退出饥饿模式。

上面是对照代码的理解过程，下面我们总结下：
正常情况下, 当一个Goroutine获取到锁后, 其他的G获取锁时会自旋或者进入睡眠状态，等待信号量唤醒，自旋是希望当前的G能获取到锁，因为它持有CPU，
可以避免资源切换。但是又不能一直自旋， 所以mutex就会有饥饿模式，如果一个G被唤醒过后, 仍然没有拿到锁, 那么该G会放在等待队列的最前面。
并且那些等待超过1ms的G还没有获取到锁，该G就会进入饥饿状态。该G是饥饿状态并且Mutex是Locked状态时，才有可能给Mutex设置成饥饿状态。

获取到锁的G解锁时，将mutex的状态设为unlock，然后发出信号，等待的G开始抢夺锁的。但是如果mutex处于饥饿状态，就会将信号发给第一个G，唤醒它。这就是G排队。


#### 解锁过程
```go
// Unlock unlocks m.
// It is a run-time error if m is not locked on entry to Unlock.
//
// A locked Mutex is not associated with a particular goroutine.
// It is allowed for one goroutine to lock a Mutex and then
// arrange for another goroutine to unlock it.
func (m *Mutex) Unlock() {
	if race.Enabled {
		_ = m.state
		race.Release(unsafe.Pointer(m))
	}

	// Fast path: drop lock bit.
	new := atomic.AddInt32(&m.state, -mutexLocked)
	if new != 0 {
		// Outlined slow path to allow inlining the fast path.
		// To hide unlockSlow during tracing we skip one extra frame when tracing GoUnblock.
		m.unlockSlow(new)
	}
}

func (m *Mutex) unlockSlow(new int32) {
	if (new+mutexLocked)&mutexLocked == 0 {
		throw("sync: unlock of unlocked mutex")
	}
	if new&mutexStarving == 0 {
		old := new
		for {
			// If there are no waiters or a goroutine has already
			// been woken or grabbed the lock, no need to wake anyone.
			// In starvation mode ownership is directly handed off from unlocking
			// goroutine to the next waiter. We are not part of this chain,
			// since we did not observe mutexStarving when we unlocked the mutex above.
			// So get off the way.
			if old>>mutexWaiterShift == 0 || old&(mutexLocked|mutexWoken|mutexStarving) != 0 {
				return
			}
			// Grab the right to wake someone.
			new = (old - 1<<mutexWaiterShift) | mutexWoken
			if atomic.CompareAndSwapInt32(&m.state, old, new) {
				runtime_Semrelease(&m.sema, false, 1)
				return
			}
			old = m.state
		}
	} else {
		// Starving mode: handoff mutex ownership to the next waiter, and yield
		// our time slice so that the next waiter can start to run immediately.
		// Note: mutexLocked is not set, the waiter will set it after wakeup.
		// But mutex is still considered locked if mutexStarving is set,
		// so new coming goroutines won't acquire it.
		runtime_Semrelease(&m.sema, true, 1)
	}
}
```
解除mutex的lock状态，获取到最新的状态new。如果new是饥饿状态，唤醒第一个G，获取锁。解锁完成。
如果不是饥饿状态。当前没有G等待，或者有G已经被唤醒去加锁了。就不需要做唤醒的动作。退出即可。
否则将等待的G-1，并一直尝试将G设置为唤醒状态，释放信号量，通知所有的G都可以去抢夺锁。设置成功解锁完成，否则继续执行。

### 引申问题

#### 源码阅读注意的点 
在理解sync.mutex的时候，一定要注意的点是，m的state在代码执行过程中，很可能会有其他的G改变其状态。所以查看代码逻辑的时候，
一定要时刻记得这个点
#### woken状态的意义
Woken状态用于加锁和解锁过程的通信，举个例子，同一时刻，两个协程一个在加锁，一个在解锁，在加锁的协程可能在自旋过程中，
此时把Woken标记为1，用于通知解锁协程不必释放信号量了，好比在说：你只管解锁好了，不必释放信号量，我马上就拿到锁了。

#### 为什么重复解锁会panic
Unlock的逻辑是，解除mutex的lock状态，然后检查是否有协程等待，有的话释放信号量，唤醒协程。如果多次unlock的话，就会
发送多个信号量，唤醒多个G去抢夺锁。会造成不必要的竞争，也会造成协程切换，浪费资源，实现复杂度也会增加。

#### G的饥饿和mutex饥饿的关系
只有G处于饥饿状态的时候，才会将mutex设为饥饿状态。当mutex处于饥饿状态时，才可能会让饥饿的G获取到锁。需要注意的是，设mutex
为饥饿状态的G不一定会获取到锁，有可能会被其他G截胡。

#### G可以成功获取锁的情况
 - 第一次加锁的时候m.state=0，一定是可以获取锁。没有其他的G获取锁，没有改变其状态。
 - 当前的mutex不是饥饿状态，也不是lock状态，尝试CAS加锁的时候，如果没有其他G改变m状态，可以成功。
 - 某个G被唤醒后，重新获取mutex，此时mutex处于饥饿状态，没有其他的G来抢夺，因为这个时候只唤醒了饥饿的G，G也可以成功。

### 参考资料
- https://blog.csdn.net/baolingye/article/details/111357407
- https://blog.csdn.net/qq_37102984/article/details/115322706
- https://mp.weixin.qq.com/s/BZvfNn_Vre7o2T8BZ4LLMw