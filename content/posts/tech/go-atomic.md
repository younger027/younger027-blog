---
title: "Go Atomic" #标题
date: 2023-03-13T16:40:32+08:00 #创建时间
lastmod: 2023-03-13T16:40:32+08:00 #更新时间
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

### 源码解析
源码版本：1.18

atomic包主要支持一些原子操作，首先我们来看看源码doc文件(atomic的源码路径：/src/runtime/internal/atomic)对atomic的介绍：

```go
/*
Package atomic provides atomic operations, independent of sync/atomic,
to the runtime.

On most platforms, the compiler is aware of the functions defined
in this package, and they're replaced with platform-specific intrinsics.
On other platforms, generic implementations are made available.

Unless otherwise noted, operations defined in this package are sequentially
consistent across threads with respect to the Vals they manipulate. More
specifically, operations that happen in a specific order on one thread,
will always be observed to happen in exactly that order by another thread.
*/

/*
包atomic提供原子操作，独立于sync/atomic。向运行时提供原子操作。

在大多数平台上，编译器都知道这个包中定义的函数。
在这个包中定义的函数，它们被替换成特定平台的本征。
在其他平台上，通用的实现是可用的。

除非另有说明，本包中定义的操作在顺序上是
在它们所操作的值方面，在不同的线程中是一致的。更具体地说
具体来说，在一个线程上以特定顺序发生的操作。
将总是被另一个线程观察到以该顺序发生。
*/

package atomic
```
atomic 提供的是原子操作，atomic 包中支持六种类型

- int32
- uint32
- int64
- uint64
- uintptr
- unsafe.Pointer

每一个类型都支持以下操作：
```go
//以Int32类型举例

// SwapInt32 atomically stores new into *addr and returns the previous *addr Val.
func SwapInt32(addr *int32, new int32) (old int32);

// SwapInt32 atomically stores new into *addr and returns the previous *addr Val.
// 原子性的比较*addr和old，如果相同则将new赋值给*addr并返回真。
func CompareAndSwapInt32(addr *int32, old, new int32) (swapped bool);

// AddInt32 atomically adds delta to *addr and returns the new Val.
func AddInt32(addr *int32, delta int32) (new int32);

// LoadInt32 atomically loads *addr.
func LoadInt32(addr *int32) (val int32);

// StoreInt32 atomically stores val into *addr.
func StoreInt32(addr *int32, val int32);

```

其中大部分的函数都是由汇编代码实现。源码包中根据系统平台的不同实现了不同的.s汇编代码。具体可查看/src/runtime/internal/atomic下的代码，我们举例
说明下amd64指令架构下的汇编实现：
```
// bool Cas(int32 *val, int32 old, int32 new)
// Atomically:
//	if(*val == old){
//		*val = new;
//		return 1;
//	} else
//		return 0;
TEXT ·Cas(SB),NOSPLIT,$0-17
	MOVQ	ptr+0(FP), BX
	MOVL	old+8(FP), AX
	MOVL	new+12(FP), CX
	LOCK
	CMPXCHGL	CX, 0(BX)
	SETEQ	ret+16(FP)
	RET
```
Cas函数主要对比val地址中的值是否和old一样，如果一样val赋值new，返回true，否则完成false。对应的汇编代码解释如下：
这段代码使用了汇编语言来进行内存操作，下面是对每一行代码的解释：

1. MOVQ ptr+0(FP), BX：将变量 ptr 在帧指针 FP 上的偏移量为 0 的位置的值，即 ptr 所指向的内存地址的值，传送到寄存器 BX 中。

2. MOVL old+8(FP), AX：将变量 old 在帧指针 FP 上的偏移量为 8 的位置的值，即 old 所指向的内存地址的值，传送到寄存器 AX 中。

3. MOVL new+12(FP), CX：将变量 new 在帧指针 FP 上的偏移量为 12 的位置的值，即 new 所指向的内存地址的值，传送到寄存器 CX 中。

4. LOCK：该指令是一个前缀指令，作用是在多核心处理器中保证对内存的原子性操作，即在对内存进行操作时，不会被其他处理器的操作中断。

5. CMPXCHGL CX, 0(BX)：该指令是一个比较并交换指令，作用是将内存地址 BX 上的值与 AX 进行比较，如果相等，则将 CX 替换为内存地址 BX 上的值，
并返回成功的标记；如果不相等，则什么都不做，返回失败的标记。(关于CMPXCHGL指令的操作数，它有一个默认的操作者，即eax寄存器。也就是说，当CMPXCHGL指令执行时，
它会默认使用eax寄存器作为操作数，同时，它也需要至少一个内存地址作为操作数之一)

6. SETEQ ret+16(FP)：如果比较并交换成功，则将变量 ret 在帧指针 FP 上的偏移量为 16 的位置的值设置为 1（标记成功；
如果比较并交换失败，则该指令什么也不做，即变量 ret 的值仍旧为 0。

7. RET：返回函数并结束执行

已上就是cas函数的实现过程。有几个点我们需要注意下。
1. LOCK：是一个指令前缀，其后必须跟一条“读-改-写”性质的指令，它们可以是ADD, ADC, AND, BTC, BTR, BTS, CMPXCHG, CMPXCH8B, CMPXCHG16B, 
DEC, INC, NEG, NOT, OR, SBB, SUB, XOR, XADD, XCHG。该指令是一种锁定协议，用于封锁总线，禁止其他CPU对内存的操作来保证原子性。

2. CMPXCHGL指令并不是个原子指令，在多核cpu下，还是需要lock这个指令来锁定的。lock之后的指令，会阻塞其他cpu的指令执行。具体实现是物理级别的总线锁，
cpu在硬件层面支持原子操作。但是这个锁粒度非常大，很影响执行的效率。所以MESI协议诞生，它主要是解决多核cpu时对共享数据的访问控制。 尽管有LOCK前缀，
但如果对应数据已经在cache line里，也就不用锁定总线，仅锁住缓存行即可。(这里还涉及cpu缓存的知识可以学习耗子哥的这篇文章[与程序员相关的CPU缓存知识](https://coolshell.cn/articles/20793.html))


### atomic.Val
下面主要介绍下atomic.Val这个可以存储任何数据的函数。
```go
// A Val provides an atomic load and store of a consistently typed Val.
// The zero Val for a Val returns nil from Load.
// Once Store has been called, a Val must not be copied.
//
// A Val must not be copied after first use.
type Val struct {
    v any
}

// ifaceWords is interface{} internal representation.
type ifaceWords struct {
    typ  unsafe.Pointer
    data unsafe.Pointer
}
```

Val结构就是个interface类型，实际的数据存储是ifaceWords类型。typ指针保存存储对象的类型。data指针就是实际的数据。下面我们看下Load和Store方法的实现：
```go

func (v *Val) Store(x interface{}) {
        // x为nil，直接panic
    if x == nil {
        panic("sync/atomic: store of nil Val into Val")
    }
    // 将现有的值和要写入的值转换为ifaceWords类型，这样下一步就能获取到它们的原始类型和真正的值
    vp := (*ifaceWords)(unsafe.Pointer(v))
    xp := (*ifaceWords)(unsafe.Pointer(&x))
    for {
        // 获取现有的值的type
        typ := LoadPointer(&vp.typ)
        // 如果typ为nil说明这是第一次调用Store
        if typ == nil {
            // 如果是第一次调用，就占住当前的processor，不允许其他goroutine再抢，
			// runtime_procPin方法会先获取当前goroutine
            runtime_procPin()
            // 使用CAS操作，先尝试将typ设置为^uintptr(0)这个中间状态
            // 如果失败，则证明已经有别的goroutine抢先完成了赋值操作
            // 那它就解除抢占锁，继续循环等待
            if !CompareAndSwapPointer(&vp.typ, nil, unsafe.Pointer(^uintptr(0))) {
                runtime_procUnpin()
                continue
            }
            // 如果设置成功，就原子性的更新对应的指针，最后解除抢占锁
            StorePointer(&vp.data, xp.data)
            StorePointer(&vp.typ, xp.typ)
            runtime_procUnpin()
            return
        }
        // 如果typ为^uintptr(0)说明第一次写入还没有完成，继续循环等待
        if uintptr(typ) == ^uintptr(0) {
            continue
        }
        // 如果要写入的类型和现有的类型不一致，则panic
        if typ != xp.typ {
            panic("sync/atomic: store of inconsistently typed Val into Val")
        }
        // 更新data，跳出循环
        StorePointer(&vp.data, xp.data)
        return
    }
}

```
上面代码讲解的非常清晰，简单来说，所谓的atomic.Val就像是interface一样，存储对象的类型和数据。然后通过指针，赋值操作管理对象。上面代码中有几个点需要注意：

1. runtime_procPin、runtime_procUnpin函数
在 Golang 的运行时中，每个 goroutine 都是从可用的逻辑处理器中获取执行资源，它能够随时在这些逻辑处理器之间切换。但是在某些情况下，
我们需要将一个特定的 goroutine 固定在指定的当前线程上(M)运行，而 runtime_procPin 函数就是为了实现这一功能而设计的。具体实现时，
runtime_procPin 函数会将当前 goroutine 绑定到指定的线程上，并将该线程置于固定调度模式下。这样可以确保指定的线程一直被用于执行该goroutine，
而不会被其他 goroutine 抢占。在运行完成后，使用 runtime_procUnpin 函数解除绑定。

2. unsafe.Pointer(^uintptr(0))只是个中间状态，并发store时，用来判断初始化写入是否完成。


```go
func (v *Val) Load() (x interface{}) {
        // 将*Val指针类型转换为*ifaceWords指针类型
    vp := (*ifaceWords)(unsafe.Pointer(v))
    // 原子性的获取到v的类型typ的指针
    typ := LoadPointer(&vp.typ)
    // 如果没有写入或者正在写入，先返回，^uintptr(0)代表过渡状态，这和Store是对应的
    if typ == nil || uintptr(typ) == ^uintptr(0) {
        return nil
    }
    // 原子性的获取到v的真正的值data的指针，然后返回
    data := LoadPointer(&vp.data)
    xp := (*ifaceWords)(unsafe.Pointer(&x))
    xp.typ = typ
    xp.data = data
    return
}
```
Load代码就比较简单了，对比Store相信你肯定能完全理解。



### 参考
- https://learnku.com/docs/learngo/1.0/atomic/14038
- https://zhuanlan.zhihu.com/p/343563035
- https://www.cyub.vip/2021/04/05/Golang%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%E7%B3%BB%E5%88%97%E4%B9%8Batomic%E5%BA%95%E5%B1%82%E5%AE%9E%E7%8E%B0/
- https://blog.csdn.net/lotluck/article/details/78793468
- https://coolshell.cn/articles/20793.html