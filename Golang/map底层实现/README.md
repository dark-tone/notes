# 底层代码
<img src="https://raw.githubusercontent.com/dark-tone/notes/main/Golang/imgs/4.jpg" width="700px">

``` go
type hmap struct {
	// Note: the format of the hmap is also encoded in cmd/compile/internal/gc/reflect.go.
	// Make sure this stays in sync with the compiler's definition.
	count     int // # live cells == size of map.  Must be first (used by len() builtin)
	flags     uint8
	B         uint8  // log_2 of # of buckets (can hold up to loadFactor * 2^B items)
	noverflow uint16 // approximate number of overflow buckets; see incrnoverflow for details
	hash0     uint32 // hash seed

	buckets    unsafe.Pointer // array of 2^B Buckets. may be nil if count==0.
	oldbuckets unsafe.Pointer // previous bucket array of half the size, non-nil only when growing
	nevacuate  uintptr        // progress counter for evacuation (buckets less than this have been evacuated)

	extra *mapextra // optional fields
}
```
**buckets**：指向哈希桶的指针。<br>
**oldbuckets**： 是当桶扩容时指向旧桶的指针（只有扩容时不为nil，扩容机制类似于redis）。<br>
**nevacuate**： 是当桶进行调整时指示的搬迁进度, 小于此地址的 buckets 是以前搬迁完毕的哈希桶。

<br>

``` go
type bmap struct {
    topbits  [8]uint8
    keys     [8]keytype
    elems    [8]elemtype
    overflow uintptr
}
```
bmap 表征了 go map 哈希桶的结构, 其中 topbits 是键哈希值的高 8 位, keys 存放了哈希桶中所有键, elems 存放了哈希桶中的所有值, overflow 是一个 uintptr 类型指针, 存放了所指向的溢出桶的地址, go map 的每个哈希桶最多存放 8 个键值对, 当经由哈希函数映射到该地址的元素数超过 8 个时, 会将新的元素放到溢出桶中, 并使用 overflow 指针链向这个溢出桶。

<br>

``` go
// mapextra holds fields that are not present on all maps.
// 用于处理key和elem不含指针，gc会释放overflow的问题（联系三色标记法）
type mapextra struct {
	// If both key and elem do not contain pointers and are inline, then we mark bucket
	// type as containing no pointers. This avoids scanning such maps.
	// However, bmap.overflow is a pointer. In order to keep overflow buckets
	// alive, we store pointers to all overflow buckets in hmap.extra.overflow and hmap.extra.oldoverflow.
	// overflow and oldoverflow are only used if key and elem do not contain pointers.
	// overflow contains overflow buckets for hmap.buckets.
	// oldoverflow contains overflow buckets for hmap.oldbuckets.
	// The indirection allows to store a pointer to the slice in hiter.
	overflow    *[]*bmap
	oldoverflow *[]*bmap

	// nextOverflow holds a pointer to a free overflow bucket.
	nextOverflow *bmap
}
```
当一个 map 的 key 和 elem 都不含指针并且他们的长度都没有超过 128 时(当 key 或 value 的长度超过 128 时, go 在 map 中会使用指针存储), 该 map 的 bucket 类型会被标注为不含有指针, 这样 gc 不会扫描该 map, 这会导致一个问题, bucket 的底层结构 bmap 中含有一个指向溢出桶的指针(uintptr类型, uintptr指针指向的内存不保证不会被 gc free 掉), 当 gc 不扫描该结构时, 该指针指向的内存会被 gc free 掉, 因此在 hmap 结构中增加了 mapextra 字段, 其中 overflow 是一个指向保存了所有 hmap.buckets 的溢出桶地址的 slice 的指针, 相对应的 oldoverflow 是指向保存了所有 hmap.oldbuckets 的溢出桶地址的 slice 的指针, 只有当 map 的 key 和 elem 都不含指针时这两个字段才有效, 因为这两个字段设置的目的就是避免当 map 被 gc 跳过扫描带来的引用内存被 free 的问题, 当 map 的 key 和 elem 含有指针时, gc 会扫描 map, 从而也会获知 bmap 中指针指向的内存是被引用的, 因此不会释放对应的内存。

# 查找
哈希函数：使用了 aes 和 memhash 两类哈希, 当运行架构支持 aes 哈希时会优先使用 aes 作为 HashFunc。

1. 使用 key 和哈希表的 hash0, 做哈希函数运算得到哈希值。
2. 根据B值得知哈希桶数量，取余得出对应的哈希桶位置。
3. 基于哈希值的高 8 位与桶中的 topbits 依次比较, 若相等便可以根据 topbits 所在的相对位置计算出 key 所在的相对位置, 进一步比较 key 是否相等, 若 key 相等则此次查找过程结束, 返回对应位置上 elem, 若 key 不相等, 则继续往下比较 topbits, 若当前桶中的所有 topbits 均与此次要找到的元素的 key 的哈希值的高 8 位不相等, 则继续沿着 overflow 向后探查溢出桶, 重复刚刚的过程, 直到找到对应的 elem, 或遍历完所有的溢出桶仍未找到目标元素, 此时返回该类型的零值。
 
# 扩容
触发条件：
- 装载因子过大，即键值对太多了。（loadFactory=element.Length/hashTable.length）
- 溢出桶过多。（先插入，后删除，导致生成许多多余的溢出桶，键值对变得松散。**注意此情况触发的是等量扩容**）

go 扩容的基本步骤是首先根据扩容条件(装载因子 >= 6.5 或 溢出桶数目太多), 而确定扩容后的大小, 然后创建该大小的新哈希桶, 这时会将 hmap 中的 buckets 指针指向新创建的哈希桶, 而原先的哈希桶地址则保存在 oldbuckets 指针中, 该段逻辑位于 src/runtime/map/go#hashGrow, 该函数只是用于为新的哈希桶创建存储空间, 并未开始搬迁, 具体的搬迁逻辑位于 src/runtime/map.go#evacuate 中, 若是因为溢出桶数目过多造成的扩容, 则扩容是等量扩容, 整个过程是将原 Bucket 中的所有元素迁移到新的等量的 Bucket 中, 在迁移的过程中, 哈希桶(非溢出桶)的相对位置不会发生改变, 即原先位于 N 号 Bucket 的元素会映射到新的 N 号 Bucket 位置上, 而若是翻倍扩容, 则元素会被平均(此处不是数学意义上的严格平均, 其具体分流逻辑是用哈希值与原 Bucket 数目做逻辑与运算, 取决于 HashFunc 的该位是否足够平均)分流到两段上, 在 go 中每次只搬迁两个 Bucket, 当所有元素都搬迁完毕之后, hmap 的 oldbuckets 指针会被设置为 nil, 因此 oldbuckets 指针是否为 nil 可以作为当前 map 是否处于扩容状态的一个标志。

# 参考资料
[年度最佳【golang】map详解](https://segmentfault.com/a/1190000023879178)

[Go 语言 map 的底层实现完整剖析](https://zhuanlan.zhihu.com/p/406751292)

[Golang Map 底层实现](https://blog.csdn.net/jacson__/article/details/124742748)