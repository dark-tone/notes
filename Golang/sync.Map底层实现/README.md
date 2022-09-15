# 适用场景
sync.map是用读写分离实现的，其思想是空间换时间。和map+RWLock的实现方式相比，它做了一些优化：可以无锁访问read map，而且会优先操作read map，倘若只操作read map就可以满足要求(增删改查遍历)，那就不用去操作write map(它的读写都要加锁)，所以在某些特定场景中它发生锁竞争的频率会远远小于map+RWLock的实现方式。

适用于**读多写少**的情况，如果并发写多，用map+RWLock会更好一些。

# 相关结构
``` go
type Map struct {
	mu Mutex
	read atomic.Value // readOnly
    // dirty == nil的情况：1.被初始化 2.提升为read后，但它不能一直为nil，否则read和dirty会数据不一致。
	dirty map[interface{}]*entry
    // 穿透访问dirty的次数,若miss等于dirty的长度，dirty会提升成read
	misses int
}

type readOnly struct {
  m       map[interface{}]*entry
  amended bool // amended等于true代表dirty中有readOnly.m中不存在的entry
}

type entry struct {
       // p == nil：entry已从readOnly中删除但存在于dirty中
       // p == expunged：entry已从Map中删除且不在dirty中
       // p == 其他值：entry为正常值
       p unsafe.Pointer // *interface{}
}
```

# Store
``` go
func (m *Map) Store(key, value interface{}) {
    // 把m.read转成结构体readOnly
    read, _ := m.read.Load().(readOnly)
    // 若key在readOnly.m中且entry.p不为expunged（没有标记成已删除）即key同时存在于readOnly.m和dirty
    // ,用CAS技术更新value 【注】：e.tryStore在entry.p == expunged时会立刻返回false，否则用CAS
    // 尝试更新对应的value, 更新成功会返回true
    if e, ok := read.m[key]; ok && e.tryStore(&value) {
        return
    }
    // key不存在于readOnly.m或者entry.p==expunged(entry被标记为已删除)，加锁访问dirty
    m.mu.Lock()
    // 双重检测：若加锁前Map.dirty被提升为readOnly，则前面的read.m[key]可能无效，所以需要再次检测key是
    // 否存在于readOnly中
    read, _ = m.read.Load().(readOnly)
    // 若key在于readOnly.m中
    if e, ok := read.m[key]; ok {
        // entry.p之前的状态是expunged，把它置为nil
        if e.unexpungeLocked() {
            // 之前dirty中没有此key，所以往dirty中添加此key  
            m.dirty[key] = e
        }
        // 更新（把value的地址原子赋值给指针entry.p）
        e.storeLocked(&value)  
      // 若key在dirty中
    } else if e, ok := m.dirty[key]; ok { 
        // 更新（把value的地址原子赋值给指针entry.p）
        e.storeLocked(&value)
      // 来了个新key
    } else { 
        // dirty中没有新数据，往dirty中添加第一个新key
        if !read.amended {  
            // 把readOnly中未标记为删除的数据拷贝到dirty中
            m.dirtyLocked()  
            // amended:true，因为现在dirty有readOnly中没有的key
            m.read.Store(readOnly{m: read.m, amended: true})
        }
        // 把这个新的entry加到dirty中
        m.dirty[key] = newEntry(value)
    }
    m.mu.Unlock()
}
```
- Store方法优先无锁访问readOnly，未命中会加锁访问dirty。
- Store方法中的双重检测机制在下面的Load、Delete、Range方法中都会用到，原因是：加锁前Map.dirty可能已被提升为Map.read，所以加锁后还要再次检查key是否存在于Map.read中。
- dirtyLocked方法在dirty为nil（刚被提升成readOnly或者Map初始化时）会从readOnly中拷贝数据，如果readOnly中数据量很大，可能偶尔会出现**性能抖动**。
- sync.map不适合用于频繁插入新key-value的场景，因为此操作会频繁加锁访问dirty会导致性能下降。更新操作在key存在于readOnly中且值没有被标记为删除(expunged)的场景下会用无锁操作CAS进行性能优化，否则也会加锁访问dirty。


# Load
``` go
func (m *Map) Load(key interface{}) (value interface{}, ok bool) {
    // 从m.read中换出readOnly，然后从里面找key，这个过程不加锁
    read, _ := m.read.Load().(readOnly)
    e, ok := read.m[key]

    // readOnly中不存在此key但Map.dirty可能存在
    if !ok && read.amended {
        // 加锁访问Map.dirty
        m.mu.Lock()
        // 双重检测:若加锁前Map.dirty被替换为readonly，则前面m.read.Load().(readOnly)无效，需         
        // 要再次检查
        read, _ = m.read.Load().(readOnly)
        e, ok = read.m[key]
        // read.m没有此key && dirty里有可能有（dirty中有read.m没有的数据）
        if !ok && read.amended {
            // 从dirty中获取key对应的entry
            e, ok = m.dirty[key]
            // 无论Map.dirty中是否有这个key，miss都加一，若miss大小等于dirty的长度，dirty中的元素会被
            // 加到Map.read中
            m.missLocked()
        }
        m.mu.Unlock()
    }
    if !ok {
        return nil, false
    }
    // 若entry.p被删除（等于nil或expunged）返回nil和不存在（false），否则返回对应的值和存在（true）
    return e.load()
}
```
- Load方法会优先无锁访问readOnly，未命中后如果Map.dirty中可能存在这个数据就会加锁访问Map.dirty。
- Load方法如果访问readOnly中不存在但dirty中存在的key，就要加锁访问Map.dirty从而带来额外开销。



# Delete
``` go
func (m *Map) LoadAndDelete(key interface{}) (value interface{}, loaded bool) {
        // 从m.read中换出readOnly，然后从里面找key，此过程不加锁
  read, _ := m.read.Load().(readOnly)
  e, ok := read.m[key]

        // readOnly不存在此key，但dirty中可能存在
  if !ok && read.amended {
                // 加锁访问dirty
    m.mu.Lock()
                // 双重检测:若加锁前Map.dirty被替换为readonly，则前面m.read.Load().(readOnly)无 
                // 效，需要再次检查
    read, _ = m.read.Load().(readOnly)
    e, ok = read.m[key]
                // readOnly不存在此key，但是dirty中可能存在
    if !ok && read.amended {
      e, ok = m.dirty[key]
      delete(m.dirty, key)
      m.missLocked()
    }
    m.mu.Unlock()
  }
  if ok {
                // 如果entry.p不为nil或者expunged，则把entry.p软删除（标记为nil）
    return e.delete()
  }
  return nil, false
}
```
- 删除readOnly中存在的key，可以不用加锁。
- 如果删除readOnly中不存在的或者Map中不存在的key，都需要加锁。

# Range
``` go
func (m *Map) Range(f func(key, value interface{}) bool) { 
     read, _ := m.read.Load().(readOnly) 
     if read.amended { // dirty存在readOnly中不存在的元素 
         // 加锁访问dirty
         m.mu.Lock() 
         // 再次检测read.amended，因为加锁前它可能已由true变成false
         read, _ = m.read.Load().(readOnly) 
         if read.amended { 
             // readOnly.amended被默认赋值成false 
             read = readOnly{m: m.dirty} 
             m.read.Store(read) 
             m.dirty = nil 
             m.misses = 0 
        } 
        m.mu.Unlock() 
    } 
    // 遍历readOnly.m
    for k, e := range read.m { 
         v, ok := e.load() 
         if !ok {
             continue 
         } 
         if !f(k, v) { 
             break 
         } 
    } 
}
```
- Range方法Map的全部key都存在于readOnly中时，是无锁遍历的，性能最高。
- Range方法在readOnly只存在Map中的部分key时，会一次性加锁拷贝dirty的元素到readOnly，减少多次加锁访问dirty中的数据。

# 参考资料
[sync.Map详解](https://blog.csdn.net/a348752377/article/details/104972194)

[浅谈Golang两种线程安全的map](https://zhuanlan.zhihu.com/p/449078860)

[sync.Map详解](https://blog.csdn.net/weixin_41335923/article/details/124061082)