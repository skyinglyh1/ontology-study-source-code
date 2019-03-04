## types
#### StoreIterator interface{
```
First() bool
Next() bool
Key() []byte
Value() []byte   //Return the current item key
Release()
Error() 
```

}
#### PersistStore interface{
```
Put(key []byte, value []byte) error
Get(key []byte) ([] byte, error)
Has(key []byte) (bool, error)
Delete(key []byte) error
NewBatch()  // start commit batch
BatchPut(key []byte, value []byte)      // put a key-value pair to BatchPut
BatchDelete(key []byte)                 // delete the key in batch
BatchCommit() error                     // commit batch to store 
Close() error                           // close store
NewIterator(prefix []byte) StoreIterator //Return the iterator of store
```
}


#### StateStore interface{
```
//Add key-value pair to store
TryAdd(prefix DataEntryPrefix, key []byte, value states.StateValue)
//Get key from state store, if not exist, add it to store
TryGetOrAdd(prefix DataEntryPrefix, key []byte, value states.StateValue) error
//Get key from state store
TryGet(prefix DataEntryPrefix, key []byte) (*StateItem, error)
//Delete key in store
TryDelete(prefix DataEntryPrefix, key []byte)
//iterator key in store
Find(prefix DataEntryPrefix, key []byte) ([]*StateItem, error)
```
}

#### MemoryCacheStore interface {
```
	//Put the key-value pair to store
	Put(prefix byte, key []byte, value states.StateValue, state ItemState)
	//Get the value if key in store
	Get(prefix byte, key []byte) *StateItem
	//Delete the key in store
	Delete(prefix byte, key []byte)
	//Get all updated key-value set
	GetChangeSet() map[string]*StateItem
	// Get all key-value in store
	Find() []*StateItem
```
}


###### /core/store/leveldbstore/leveldb_store.go
type LevelDBStore struct {
	db    *leveldb.DB // LevelDB instance
	batch *leveldb.Batch
}
```leveldb_store.go``` has implemented the the following method,
```
Put(key []byte, value []byte) error
Get(key []byte) ([] byte, error)
Has(key []byte) (bool, error)
Delete(key []byte) error
NewBatch()  // start commit batch
BatchPut(key []byte, value []byte)      // put a key-value pair to BatchPut
BatchDelete(key []byte)                 // delete the key in batch
BatchCommit() error                     // commit batch to store 
Close() error                           // close store
NewIterator(prefix []byte) StoreIterator //Return the iterator of store
```
, therefore, we can see the ```LevelDBStore``` is the ```PersistStore```.

###### /core/store/overlaydb/memdb.go



MemDB is an in-memdb key/value database.
type MemDB struct {
	cmp comparer.BasicComparer
	rnd *rand.Rand

	kvData []byte
	// Node data:
	// [0]         : KV offset
	// [1]         : Key length
	// [2]         : Value length
	// [3]         : Height
	// [3..height] : Next nodes
	nodeData  []int
	prevNode  [tMaxHeight]int
	maxHeight int
	n         int
	kvSize    int
}
```MemDB``` struct has implemented skip-list algorithm, including the following methods,
```
randHeight() (int) // given maxHeight = 12, find a random height
findGE(key []byte, prev bool) (int, bool) 
findLT(key []byte) int
findLast() int
Put(key []byte, value []byte)
Delete(key []byte)
Get(key []byte) (value []byte, unknown bool)
Find(key []byte) (rkey, value [] byte, err, error)
ForEach(f func(key, value []byte))
NewIterator(slice *util.Range) iterator.Iterator
Capacity() int
Size() int
Free() int
Len() int
Reset() 
NewMemDB(capicity int, kvNum int)```




```dbIter``` struct actually contains ```StoreIterator``` and add some other elements.
Here is the what ```dbIter``` defines,
type dbIter struct {
	util.BasicReleaser
	p          *MemDB
	slice      *util.Range
	node       int
	forward    bool
	key, value []byte
	err        error
}. 

And ```dbIter``` implements the following methods.

```fill(checkStart, checkLimit bool) bool
valid() bool
First() bool
Last() bool
Seek(key []byte) bool
Next() bool 
Prev() bool
Key() []byte
Value() []byte
Error() error
Release() 
```

###### core/store/overlaydb/overlaydb.go
type OverlayDB struct {
	store common.PersistStore
	memdb *MemDB
	dbErr error
}

```OverlayDB``` has implemented the following methods.
```
NewOverlayDB(store common.PersistStore) * OverlayDB
Reset() 
Error()
SetError()
Get(key []byte) (value []byte)
Put(key []byte, value []byte)
Delete(key []byte)
CommitTo()
GetWriteSet() *MemDB
ChangeHash() comm.Uint256
NewIterator(key []byte) common.StoreIterator
```
overlaydb_test.go has implement 
```
TestNewOverlayDB() to initialize levelDB, which is used to initialize a new overlayDB. Through overlayDB operation like Put, or Get operation (actually through memDB operations), we modify the contents of the ledger which is overlayDB (actually levelDB).
BenchmarkOverlayDBSerialPut-8   	      20	  55750955 ns/op
BenchmarkStateBatch-8   	              20	  62534540 ns/op
BenchmarkOverlayDBRandomPut-8   	      20	  70561320 ns/op
```


###### core/store/statestore/memeory_store.go
type MemoryStore struct {
	memory map[string]*common.StateItem
}
memeory_store.go implements 
```
NewMemDatabase() * MemoryStore
Put(prefix byte, key []byte, value states.StateValue, state common.ItemState)
Get(prefix byte, key []byte) *common.StateItem()
Delete(prefix byte, key []byte)
Find() []*common.StateItem  
GetChangeSet() map[string]*commonStateItem
```
, which implies that MemoryStore belongs to MemoryCacheStore type.

###### core/store/statestore/state_batch.go
type StateBatch struct {
	store       common.PersistStore
	memoryStore common.MemoryCacheStore
	dbErr       error
}
```
NewStateStoreBatch(memoryStore common.MemoryCacheStore, store common.PersistStore) *StateBatch
Find(prefix common.DataEntryPrefix, key []byte) ([]*common.StateItem, error)
TryAdd(prefix common.DataEntryPrefix, key []byte, value states.StateValue)
TryGetOrAdd(prefix common.DataEntryPrefix, key []byte, value states.StateValue) error
TryGet(prefix common.DataEntryPrefix, key []byte) (*common.StateItem, error)
TryDelete(prefix common.DataEntryPrefix, key []byte) 
CommitTo() error
SetError(err error)
Error() error
```

###### smartcontract/storage/cachedb.go
在此之前，先回顾一下LRU.
据结构实现LRU（Least Recently Used）缓存机制?

1）题目要求设计并实现一个cache，使用LRU调度机制来进行存储管理。LRU调度是指当存储满时，选择最近最少使用的元素移除缓存，我们知道LRU调度最新调用或使用的对象都会放在第一个，随着时间的推移，当存储满时，排在最后个那个元素就是最长时间没有被再次使用的，就是该被移除的那个。

2）设计的数据结构要满足插入、删除和移动很容易，所有很容易想到用链表，为满足插入首节点和删除最后一个元素很简单，我们同时为链表增加首节点（head）和尾节点（tail），并使用双向链表。只需要实现对链表的三种操作：插入首节点，移除尾节点和将某个给定节点移到首位（表示最新使用）

缺点：如果有几个不符合“如果数据最近被访问过，那么将来被访问的几率也更高”的规律时，会破坏缓存，导致性能下降。如被访问频率更高的数据，而cache较小。
例如，它们对扫描读模式是没有抵抗性的。但你一次顺序读取大量的数据块时，这些数据块就会填满整个缓存空间，即使它们只是被读一次。当缓存空间满了之后，你如果想向缓存放入新的数据，那些最近最少被使用的页面将会被淘汰出去。在这种大量顺序读的情况下，我们的缓存将会只包含这些新读的数据，而不是那些真正被经常使用的数据。在这些顺序读出的数据仅仅只被使用一次的情况下，从缓存的角度来看，它将被这些无用的数据填满。

另外一个挑战是：一个缓存可以根据时间进行优化（缓存那些最近使用的页面），也可以根据频率进行优化（缓存那些最频繁使用的页面）。但是这两种方法都不能适应所有的workload。而一个好的缓存设计是能自动根据workload来调整它的优化策略。

至此，我们引入缓存机制Cache ARC算法。

type CacheDB struct {
	memdb      *overlaydb.MemDB
	backend    *overlaydb.OverlayDB
	keyScratch []byte
}
```

```
