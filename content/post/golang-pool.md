---
title: "golang Pool"
date: 2019-06-23T18:49:12+08:00
draft: true
categories: ["golang"]
tags: [""]
---

# sync.Pool是什么
高并发场景下，为了减少内存分配，降低GC压力，一种办法就是使用资源池，通过从池子中复用资源，减少资源的产生。资源池使得可以在多个goroutine之间共享资源，比如内存池、网络连接池、数据库连接池以及普通对象池等。  

golang中sync.Pool就是实现了这样一种机制的对象池，需要时我们可以从中取，使用完毕后可以将资源放回。先来瞄一眼Pool的结构：  
{{< highlight go "linenos=inline" >}}
type Pool struct {
    New: func() interface{}
}
{{< /highlight >}}
Pool对外暴露的字段只有New，是一个函数类型，在Get方法返回会为nil时(池中无对象)，New指定的函数可以生成一个新值候补给Get。

# sync.Pool的特性
sync.Pool主要有以下[特性](https://godoc.org/sync)：  

1. A Pool is a set of temporary objects(临时对象集合) that may be individually saved and retrieved.   

2. Any item stored in the Pool may be removed automatically at any time without notification. If the Pool holds the only reference when this happens, the item might be deallocated(释放).  

3. A Pool is safe for use by multiple goroutines simultaneously.  

4. Pool's purpose is to cache allocated but unused items for later reuse, relieving(减轻) pressure on the garbage collector. That is, it makes it easy to build efficient, thread-safe free lists(空闲列表). However, it is not suitable for all free lists.  

5. An appropriate use of a Pool is to manage a group of temporary items silently shared among and potentially reused by concurrent independent clients of a package. Pool provides a way to amortize(摊销) allocation overhead across many clients.  

6. An example of good use of a Pool is in the fmt package, which maintains a dynamically-sized(动态大小的) store of temporary output buffers. The store scales under load (when many goroutines are actively printing) and shrinks when quiescent(不活动的；沉寂的).  

7. On the other hand, a free list(空闲列表) maintained as part of a short-lived object is not a suitable use for a Pool, since the overhead(常开支，日常管理费) does not amortize well in that scenario. It is more efficient to have such objects implement their own free list.  

8. A Pool must not be copied after first use.

其中需要注意的是第2点，由于Pool中的对象很可能无通知地被GC掉，所以在使用时，我们没法自由控制Pool中对象的数量，这样如果用Pool实现类似连接池这种逻辑比较复杂、且有状态的资源池时是有问题的，高并发下，一旦Pool中的连接被GC，那每次连接都需要重新三次握手建立连接，代价有点大。

结合以上特点，Pool非常适合做为临时的且无状态的对象暂存，以便复用。  

# sync.Pool的使用
使用sync.Pool的典型方式如下   
{{< highlight go "linenos=inline" >}}
var pool = &sync.Pool{
    New: func() interface{} {
        return NewObject()
    }
}
pool.Get()
pool.Put(val)
{{< /highlight >}}

Get方法从对象池中获取一个对象，Put方法归还一个对象， New字段值是一个函数类型对象，当从对象池中Get对象时，如果池空，则会通过该函数重新生成一个对象在返回，所以通过Get方法返回的值不一定是资源池中的值，也可能是New生成的新对象，如果返回的值是资源池中拿到的，那在返回之前该值会被从池中删掉。从如下注释中可以看到这一点   
{{< highlight go "linenos=inline" >}}
// Get selects an arbitrary item from the Pool, removes it from the Pool, and returns it to the caller.
// Get may choose to ignore the pool and treat it as empty.
// Callers should not assume any relation between values passed to Put and the values returned by Get.
// If Get would otherwise return nil and p.New is non-nil, Get returns the result of calling p.New.
func (p *Pool) Get() interface{}

// Put adds x to the pool.
func (p *Pool) Put(x interface{})
{{< /highlight >}}

由于sync.Pool本身也是一个对象，在实践中通常在程序开始时将Pool对象除使唤为全局唯一，以下代码摘抄自containerd-shim   
{{< highlight go "linenos=inline" >}}
bufPool = sync.Pool{
    New: func() interface{} {
        return bytes.NewBuffer(nil)
    },
}
{{< /highlight >}}

# sync.Pool源码实现
## Pool

{{< highlight go "linenos=inline" >}}
go/src/sync/pool.go
// A Pool must not be copied after first use.
type Pool struct {
	noCopy noCopy

	local     unsafe.Pointer // local fixed-size per-P pool, actual type is [P]poolLocal
	localSize uintptr        // size of the local array

	// New optionally specifies a function to generate
	// a value when Get would otherwise return nil.
	// It may not be changed concurrently with calls to Get.
	New func() interface{}
}
{{< /highlight >}}

{{< highlight go "linenos=inline" >}}
// Local per-P Pool appendix.
type poolLocalInternal struct {
	private interface{}   // Can be used only by the respective P.
	shared  []interface{} // Can be used by any P.
	Mutex                 // Protects shared.
}

type poolLocal struct {
	poolLocalInternal

	// Prevents false sharing on widespread platforms with
	// 128 mod (cache line size) = 0 .
	pad [128 - unsafe.Sizeof(poolLocalInternal{})%128]byte
}

// from runtime
func fastrand() uint32

var poolRaceHash [128]uint64
{{< /highlight >}}

{{< highlight go "linenos=inline" >}}
// poolRaceAddr returns an address to use as the synchronization point
// for race detector logic. We don't use the actual pointer stored in x
// directly, for fear of conflicting with other synchronization on that address.
// Instead, we hash the pointer to get an index into poolRaceHash.
// See discussion on golang.org/cl/31589.
func poolRaceAddr(x interface{}) unsafe.Pointer {
	ptr := uintptr((*[2]unsafe.Pointer)(unsafe.Pointer(&x))[1])
	h := uint32((uint64(uint32(ptr)) * 0x85ebca6b) >> 16)
	return unsafe.Pointer(&poolRaceHash[h%uint32(len(poolRaceHash))])
}
{{< /highlight >}}

{{< highlight go "linenos=inline" >}}
// Put adds x to the pool.
func (p *Pool) Put(x interface{}) {
	if x == nil {
		return
	}
	if race.Enabled {
		if fastrand()%4 == 0 {
			// Randomly drop x on floor.
			return
		}
		race.ReleaseMerge(poolRaceAddr(x))
		race.Disable()
	}
	l := p.pin()
	if l.private == nil {
		l.private = x
		x = nil
	}
	runtime_procUnpin()
	if x != nil {
		l.Lock()
		l.shared = append(l.shared, x)
		l.Unlock()
	}
	if race.Enabled {
		race.Enable()
	}
}
// Get selects an arbitrary item from the Pool, removes it from the
// Pool, and returns it to the caller.
// Get may choose to ignore the pool and treat it as empty.
// Callers should not assume any relation between values passed to Put and
// the values returned by Get.
//
// If Get would otherwise return nil and p.New is non-nil, Get returns
// the result of calling p.New.
func (p *Pool) Get() interface{} {
	if race.Enabled {
		race.Disable()
	}
	l := p.pin()
	x := l.private
	l.private = nil
	runtime_procUnpin()
	if x == nil {
		l.Lock()
		last := len(l.shared) - 1
		if last >= 0 {
			x = l.shared[last]
			l.shared = l.shared[:last]
		}
		l.Unlock()
		if x == nil {
			x = p.getSlow()
		}
	}
	if race.Enabled {
		race.Enable()
		if x != nil {
			race.Acquire(poolRaceAddr(x))
		}
	}
	if x == nil && p.New != nil {
		x = p.New()
	}
	return x
}
{{< /highlight >}}

# 参考
[package sync](https://godoc.org/sync)
