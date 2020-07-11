---
title: "client-go 源码分析七：Informer架构"
date: 2020-07-05T21:18:51+08:00
draft: true
categories: ["技术"]
tags: ["k8s"]
---
<!--more-->
# 简介
Informer机制是`k8s.io/client-go`的核心。我们知道，kubernetes中，各个组件之间通过HTTP协议进行通信，不依赖任何中间件，消息的实时性、可靠性、顺序性就由Informer机制保证。所以说，Informer机制对于组件之间的通信至关重要。

# Informer机制的架构

# Reflector
Reflector用于监控给定的kubernetes资源，当被监控的资源状态发生变化时，会触发资源相应的Added，Updated、Deleted等事件，并将资源对象存放到本地缓存DeltaFIFO中。
{{< highlight go "linenos=inline" >}}
// k8s.io/client-go/tools/cache/reflector.go
  2 // Reflector watches a specified resource and causes all changes to be reflected in the given store.
  3 type Reflector struct {
  4     // name identifies this reflector. By default it will be a file:line if possible.
  5     name string
  6
  7     // The name of the type we expect to place in the store. The name
  8     // will be the stringification of expectedGVK if provided, and the
  9     // stringification of expectedType otherwise. It is for display
 10     // only, and should not be used for parsing or comparison.
 11     expectedTypeName string
 12     // An example object of the type we expect to place in the store.
 13     // Only the type needs to be right, except that when that is
 14     // `unstructured.Unstructured` the object's `"apiVersion"` and
 15     // `"kind"` must also be right.
 16     expectedType reflect.Type
 17     // The GVK of the object we expect to place in the store if unstructured.
 18     expectedGVK *schema.GroupVersionKind
 19     // The destination to sync up with the watch source
 20     store Store
 21     // listerWatcher is used to perform lists and watches.
 22     listerWatcher ListerWatcher
 23
 24     // backoff manages backoff of ListWatch
 25     backoffManager wait.BackoffManager
 26
 27     resyncPeriod time.Duration
 28     // ShouldResync is invoked periodically and whenever it returns `true` the Store's Resync operation is invoked
 29     ShouldResync func() bool
 30     // clock allows tests to manipulate time
 31     clock clock.Clock
  32     // paginatedResult defines whether pagination should be forced for list calls.
 31     // It is set based on the result of the initial list call.
 30     paginatedResult bool
 29     // lastSyncResourceVersion is the resource version token last
 28     // observed when doing a sync with the underlying store
 27     // it is thread safe, but not synchronized with the underlying store
 26     lastSyncResourceVersion string
 25     // isLastSyncResourceVersionGone is true if the previous list or watch request with lastSyncResourceVersion
 24     // failed with an HTTP 410 (Gone) status code.
 23     isLastSyncResourceVersionGone bool
 22     // lastSyncResourceVersionMutex guards read/write access to lastSyncResourceVersion
 21     lastSyncResourceVersionMutex sync.RWMutex
 20     // WatchListPageSize is the requested chunk size of initial and resync watch lists.
 19     // If unset, for consistent reads (RV="") or reads that opt-into arbitrarily old data
 18     // (RV="0") it will default to pager.PageSize, for the rest (RV != "" && RV != "0")
 17     // it will turn off pagination to allow serving them from watch cache.
 16     // NOTE: It should be used carefully as paginated lists are always served directly from
 15     // etcd, which is significantly less efficient and may lead to serious performance and
 14     // scalability problems.
 13     WatchListPageSize int64
 12     // Called whenever the ListAndWatch drops the connection with an error.
 11     watchErrorHandler WatchErrorHandler
 10 }
{{< /highlight >}}

# DeltaFIFO
DeltaFIFO首先是一个先进先出的FIFO队列，拥有队列的基本操作方法：Add，Update，Delete，List，Pop，Close等，Delta则是一个资源对象存储，用于保持资源对象的操作类型，操作类型有：Added，Updated，Deleted，Sync等操作类型。
{{< highlight go "linenos=inline" >}}
// k8s.io/client-go/tools/cache/delta_fifo.go
  3 // DeltaFIFO is like FIFO, but differs in two ways.  One is that the
  4 // accumulator associated with a given object's key is not that object
  5 // but rather a Deltas, which is a slice of Delta values for that
  6 // object.  Applying an object to a Deltas means to append a Delta
  7 // except when the potentially appended Delta is a Deleted and the
  8 // Deltas already ends with a Deleted.  In that case the Deltas does
  9 // not grow, although the terminal Deleted will be replaced by the new
 10 // Deleted if the older Deleted's object is a
 11 // DeletedFinalStateUnknown.
 12 //
 13 // The other difference is that DeltaFIFO has an additional way that
 14 // an object can be applied to an accumulator, called Sync.
 15 //
 16 // DeltaFIFO is a producer-consumer queue, where a Reflector is
 17 // intended to be the producer, and the consumer is whatever calls
 18 // the Pop() method.
 19 //
 20 // DeltaFIFO solves this use case:
 21 //  * You want to process every object change (delta) at most once.
 22 //  * When you process an object, you want to see everything
 23 //    that's happened to it since you last processed it.
 24 //  * You want to process the deletion of some of the objects.
 25 //  * You might want to periodically reprocess objects.
 26 //
 27 // DeltaFIFO's Pop(), Get(), and GetByKey() methods return
 28 // interface{} to satisfy the Store/Queue interfaces, but they
 29 // will always return an object of type Deltas.
 30 //
 31 // A DeltaFIFO's knownObjects KeyListerGetter provides the abilities
 32 // to list Store keys and to get objects by Store key.  The objects in
 33 // question are called "known objects" and this set of objects
 34 // modifies the behavior of the Delete, Replace, and Resync methods
 35 // (each in a different way).
 36 //
 37 // A note on threading: If you call Pop() in parallel from multiple
 38 // threads, you could end up with multiple threads processing slightly
 39 // different versions of the same object.
 40 type DeltaFIFO struct {
 41     // lock/cond protects access to 'items' and 'queue'.
 42     lock sync.RWMutex
 43     cond sync.Cond
   4     // We depend on the property that items in the set are in
  5     // the queue and vice versa, and that all Deltas in this
  6     // map have at least one Delta.
  7     items map[string]Deltas
  8     queue []string
  9
 10     // populated is true if the first batch of items inserted by Replace() has been populated
 11     // or Delete/Add/Update was called first.
 12     populated bool
 13     // initialPopulationCount is the number of items inserted by the first call of Replace()
 14     initialPopulationCount int
 15
 16     // keyFunc is used to make the key used for queued item
 17     // insertion and retrieval, and should be deterministic.
 18     keyFunc KeyFunc
 19
 20     // knownObjects list keys that are "known" --- affecting Delete(),
 21     // Replace(), and Resync()
 22     knownObjects KeyListerGetter
 23
 24     // Indication the queue is closed.
 25     // Used to indicate a queue is closed so a control loop can exit when a queue is empty.
 26     // Currently, not used to gate any of CRED operations.
 27     closed bool
 28
 29     // emitDeltaTypeReplaced is whether to emit the Replaced or Sync
 30     // DeltaType when Replace() is called (to preserve backwards compat).
 31     emitDeltaT
{{< /highlight >}}

# Indexer
`Indexer`是一个用来存储资源对象，并自带索引功能的本地缓存存储，`Reflector`从`DeltaFIFO`中消费Pop出来的资源对象将会存储(缓存)到Indexer中。Indexer与etcd集群中的数据完全一致，因此`client-go`可以很方便地从该本地存储中读取资源对象数据，无需每次通过向kube-apiserver发起请求和从远程etcd集群中读取，从而减轻对kube-apiserver和etcd的访问压力。

`k8s.io/client-go/tools/cache/store.go`中的cache就实现了Indexer接口

Indexer是一个接口，包含了Store定义的所有方法，所以Indexer实例首先它就是一个Store实例。
{{< highlight go "linenos=inline" >}}
// k8s.io/client-go/tools/cache/index.go
// Indexer extends Store with multiple indices and restricts each
// accumulator to simply hold the current object (and be empty after
// Delete).
//
// There are three kinds of strings here:
// 1. a storage key, as defined in the Store interface,
// 2. a name of an index, and
// 3. an "indexed value", which is produced by an IndexFunc and
//    can be a field value or any other string computed from the object.
type Indexer interface {
    Store
    // Index returns the stored objects whose set of indexed values
    // intersects the set of indexed values of the given object, for
    // the named index
    Index(indexName string, obj interface{}) ([]interface{}, error)
    // IndexKeys returns the storage keys of the stored objects whose
    // set of indexed values for the named index includes the given
    // indexed value
    IndexKeys(indexName, indexedValue string) ([]string, error)
    // ListIndexFuncValues returns all the indexed values of the given index
    ListIndexFuncValues(indexName string) []string
    // ByIndex returns the stored objects whose set of indexed values
    // for the named index includes the given indexed value
    ByIndex(indexName, indexedValue string) ([]interface{}, error)
    // GetIndexer return the indexers
    GetIndexers() Indexers

    // AddIndexers adds more indexers to this store.  If you call this after you already have data
    // in the store, the results are undefined.
    AddIndexers(newIndexers Indexers) error
}
{{< /highlight >}}

