# SDS（简单动态字符串）
### 概念
simple dynamic string，Redis自身实现的抽象字符串类型，Redis的默认字符串表示均用的是SDS。所有键值对的键均为SDS底层实现。（C字符串只会作为字符串字面量来使用）

### 代码实现
```c
struct sdshdr{
	// 记录buf数组已使用字节的数量
	// 等于SDS所保存字符串的长度
	int len;
	// 记录buf数组未使用字节的数量
	int free;
	// 字节数组，记录实际的字符串
	char buf[];
}
```
**示例图：**

<img src="https://raw.githubusercontent.com/dark-tone/notes/main/Redis/imgs/sds.jpg" width="532" height="220"/>

SDS遵循C字符串以空字符结尾的惯例，保存空字符的1字节空间不计算在SDS的len属性里面，分配空字符串空间以及添加到末尾的操作均为SDS函数自动完成的。遵循这一惯例的好处是：SDS可以直接重用一部分C字符串函数库里面的函数。

### 与C字符串区别
#### 1. O(1)级别获取字符串长度信息
#### 2. 杜绝缓冲区溢出
SDS记录了长度信息，调用相关API时，会对长度进行校验

#### 3. 减少修改字符串带来的内存重分配次数
**空间预分配:**<br>
在分配空间时，会根据字符串的长度，大于界限值时（比如1mb），会分配额外的空间来存储字符串。

**惰性空间释放:**<br>
字符串缩短时，字符串尾部字符去除（同时在末尾加上空字符），更改free属性值即可，不用重新分配空间。（多余的空间会保留）

#### 3. 二进制安全
所有SDS的API会以处理二进制的方式处理buf的数据，SDS可以保存文本和二进制数据，而C字符串只能保存文本数据

#### 4. 可复用部分C函数


# 链表
### 代码实现
```
// 链表
typedef struct list {
	// 表头节点
	listNode *head;
	// 表尾节点
	listNode *tail;
	// 链表所包含的节点数量
	unsigned long len;
	// 节点值复制函数
	void *(*dup) (void *ptr);
	// 节点值释放函数
	void (*free) (void *ptr);
	// 节点值对比函数
	int (*match) (void *ptr, void *key)
}

// 链表节点
typedef struct listNode {
	// 前置节点
	struct listNode *prev;
	// 后置节点
	struct listNode *next;
	// 节点的值
	void *value;
}
```
**示例图：**
<img src="https://raw.githubusercontent.com/dark-tone/notes/main/Redis/imgs/list.jpg" alt="" width="660" height="219" class="alignnone size-full wp-image-54" />


# 字典
### 代码实现
```
// 字典
typedef struct dict {
	// 类型特定函数
	dictType *type;
	// 私有数据
	void *privdata;
	// 哈希表
	dictht ht[2];
	// rehash索引，当rehash不在进行时，值为-1
	int rehashidx;
} dict;
// type：指向一个dictType结构的指针，每个dictType结构保存了一簇用于操作特定类型键值对的函数，Redis会为用途不同的字典设置不同的类型特定函数。
// privdata：保存了需要传给那些类型特定函数的可选参数。

// 哈希表
typedef struct dictht {
	// 哈希表数组
	dictEntry **table;
	// 哈希表大小
	unsigned long size;
	// 哈希表大小掩码，用于计算索引值
	// 总是等于size - 1
	unsigned long sizemask;
	// 该哈希表已有节点的数量
	unsigned long used;
} dictht;

// 哈希表节点
typedef struct dictEntry {
	// 键
	void *key;
	// 值
	union {
		void *val;
		uint64_tu64;
		int64_ts64;
	} v;
	// 指向下个哈希表节点，形成链表
	struct dictEntry *next;
} dictEntry;
```
**示意图：**<br>
<img src="https://raw.githubusercontent.com/dark-tone/notes/main/Redis/imgs/hash.jpg" alt="" width="720" height="387" />

### 哈希算法
当把新的键值对添加进字典时，会根据键值对的键计算出哈希值和索引值，然后根据索引值，将包含新键值对的哈希表节点放到哈希表数组的指定索引上面。（算法：MurmurHash）

### 键冲突
当两个键值对分配到同一个索引上时，会发生键冲突，采用**链地址法**来解决，即将相同哈希值的键值对用单向链表连接到同一个索引上。为了速度考虑，新的键值对会链接在链表的最前面。

### rehash
当键值对太多或太少，导致负载因子超出一个合理的范围时，会进行rehash（重新散列）操作。
1. 为另一哈希表ht[1]分配空间，空间大小取决于**要执行的操作**以及**当前ht[0]包含的键值对数量**
    - 扩展操作，ht[1]大小 = 第一个大于等于**ht[0].used * 2 * 2^n**的值
    - 收缩操作，ht[1]大小 = 第一个大于等于**ht[0].used * 2^n**的值
2. 将保存在ht[0]的所有键值对rehash到ht[1]上
3. 迁移完成后，释放ht[0]，将ht[1]设置为ht[0]，ht[1]新创建空白哈希表，为下一次rehash做准备
#### 自动扩展和收缩的情况
**自动扩展：**
1. 服务器没有在执行BGSAVE命令或BGREWRITEAOF命令，并且负载因子>=1
2. 服务器正在执行BGSAVE或BGREWRITEAOF命令，并且负载因子>=5
（负载因子 = 哈希表已保存节点数量used / 哈希表大小size）<br>
**自动收缩：**<br>
负载因子 <= 0.1

### 渐进式rehash
rehash的过程是渐进式的，防止对大字典rehash时影响正常服务。具体过程：
1. 为ht[1]分配空间
2. rehashidx从-1设置为0，表明rehash正在进行
3. 每次对字段执行添加、删除、查找或更新操作时，会顺带将ht[0]哈希表在rehashidx索引上的所有键值对rehash到ht[1]上，并将rehashidx + 1
4. rehash完毕后，将rehashidx设为-1，rehash成功

期间字典的删除、查找、更新操作会在两表同时执行，而新增则只会保存在ht[1]里面。

# 跳跃表
### 概念
可“跳跃”的有序链表，每个节点有指向其他节点的指针，查找效率：平均O(logN)，最坏O(N)复杂度

### 代码实现
zskiplistNode + zskiplist 两个结构定义。
```
// 跳表
typedef struct zskiplist {
    struct zskiplistNode *header, *tail;
    unsigned long length;
    int level;
} zskiplist;
// header：指向跳表表头节点
// tail：指向表尾节点
// level：记录跳表层数最大的那个节点的层数（表头节点不计）
// length：记录跳表长度，即节点数量（表头节点不计）

// 跳表节点
typedef struct zskiplistNode {
    // 成员对象
    sds ele;
    // 分值（节点按该值从小到大排序）
    double score;
    // 后退指针
    struct zskiplistNode *backward;
    struct zskiplistLevel {
        // 前进指针
        struct zskiplistNode *forward;
        // 跨度（两个节点之间的距离，可用于计算元素排位rank）
        unsigned long span;
    } level[]; // 根据幂次定律随机生成一个[1, 32]之间的值作为level数组的大小
} zskiplistNode;
```
**示例图：**<br>
<img src="https://raw.githubusercontent.com/dark-tone/notes/main/Redis/imgs/4-1.jpg" height="273" weight="701">

同一跳表节点保存的对象必须唯一，但多个节点保存的分值可以相同，分值相同的按照成员在字典序中的大小来进行排序。

跳表的查找是自顶向下查找的，如果发现当前元素的下一个元素大于要查找的值，就从下一层开始继续查找，直到找到当前元素或发现元素不存在。

# 整数集合
### 概览
当一个集合只包含整数元素，且数量不多时，会使用该结构实现。

### 代码实现
```
// 整数集合（可保存类型为int16_t、int32_t、int64_t的整数值）
typedef struct intset {
	// 编码方式
    uint32_t encoding;
	// 元素数量
    uint32_t length;
	// 保存元素的数组
    int8_t contents[];
} intset;
// 虽然contents属性声明是int8_t类型的数组，但实际contents数组不保存任何int8_t类型的值，contents数组的真正类型取决于encoding属性的值
```
**示意图：**<br>
<img src="https://raw.githubusercontent.com/dark-tone/notes/main/Redis/imgs/5-1.jpg" weight="530" height="183">

### 升级
每当要将一个新元素添加到整数集合里面，且新元素类型比整数集合现有所有类型要长时（注意是类型，并不是数值的大小），整数集合需要先进行升级。
1. 根据新元素类型，扩展底层数组空间大小，并为新元素分配空间。
2. 将原所有元素进行类型转换。
3. 将新元素添加至数组里面。

升级之后的元素，只会大于或小于当前的所有元素，当小于时会放置在数组最前面，反之则放最后面。

整数集合的底层实现为数组，这个数组以**有序、无重复**的方式保存集合元素。

**整数集合不支持降级操作。**


# 压缩列表
### 概览
当一个列表键/哈希键只包含少量列表项，并且每个列表项要么就是小整数值，要么就是长度比较短的字符串，那么Redis会使用压缩列表来作为列表键的底层实现。

### 示意图
**压缩列表**<br>
<img src="https://raw.githubusercontent.com/dark-tone/notes/main/Redis/imgs/6-1.jpg" weight="701" height="37">
- zlbytes：记录整个压缩列表占用的内存字节数
- zltail：记录表尾节点距离压缩列表起始地址有多少个字节
- zllen：记录节点数量，当值小于UINT16_MAX(65535)时为压缩列表结点数量，等于时节点的真实数量需要遍历整个压缩列表才能计算得出
- zlend：特殊值0xFF，用于标记压缩列表的末端

**列表节点**<br>
<img src="https://raw.githubusercontent.com/dark-tone/notes/main/Redis/imgs/6-2.jpg" weight="420" height="33">
- previous_entry_length：记录前一个节点的长度，属性长度可以是1字节或5字节
	> 前节点长度小于254字节，用1字节进行记录
	> 前节点长度>=254字节，用5字节记录，其中属性的第一字节会被设置为0xFE
- encoding：记录节点的content所保存的数据类型及长度（用位来进行记录）
- content：保存节点值，可以是字节数组或整数

### 连锁更新
previous_entry_length记录了前一个节点的长度，长度为1字节或5字节，如果插入或删除节点，有可能出现连锁更新情况（如在前面插入了一个值，导致1字节不足以保存前一个结点长度，后节点则需要重新分配内存，接着导致后续节点也需要连锁更新），但是这种情况并不多见（需要恰好有多个连续的、长度介于250字节至253字节的节点），且只要更新的节点数量不多，则对正常性能不会造成影响。


# 对象
### 概览
Redis不会直接使用上面提到的数据结构来实现键值对数据库，而是基于这些数据结构创建了一个对象系统（字符串、列表、哈希、集合、有序集合对象）

### 实现
```
typedef struct redisObject {
	// 类型（字符串、列表、哈希、集合、有序集合）
    unsigned type:4;
	// 编码，表明底层数据结构
    unsigned encoding:4;
    unsigned lru:LRU_BITS; /* LRU time (relative to global lru_clock) or
                            * LFU data (least significant 8 bits frequency
                            * and most significant 16 bits access time). */
    // 引用数量
	int refcount;
	// 指向底层实现数据结构的指针
    void *ptr;
} robj;
```
**示例图(raw类型字符串对象)：**<br>
<img src="https://raw.githubusercontent.com/dark-tone/notes/main/Redis/imgs/8-1.jpg" weight="794" height="165">

## 字符串对象
字符串对象编码可以是int、raw或embstr。
- int：保存的是可以用long类型表示的整数值时使用。
- raw：长度大于32字节的字符串时使用。
- embstr：长度<=32字节的字符串时使用。（专门用于保存短字符串的一种优化编码方式）

embstr内存块结构：<br>
<img src="https://raw.githubusercontent.com/dark-tone/notes/main/Redis/imgs/8-2.jpg" weight="518" height="60"><br>

另：
1. 可以用long double类型表示的浮点数在Redis中也是作为字符串值来保存的，set一个数，然后get，返回的会是字符串，因为是用字符串类型对象保存的。
2. embstr编码的字符串是只读的（因为没有这种类型的修改方法），对embstr编码的字符串进行任何修改命令时，会先将编码由embstr转换为raw。

## 列表对象
列表对象的编码可以是ziplist或linkedlist

## 哈希对象
哈希对象的编码可以是ziplist或者hashtable

## 集合对象
集合对象的编码可以是intset或者是hashtable

## 有序集合
有序集合的编码可以是ziplist或者skiplist
skiplist编码的对象使用**zset**结构作为底层实现，同时包含一个字典和一个跳跃表
```
typedef struct zset {
	zskiplist *zsl;
	dict *dict;
} zset;
```
虽然zset结构同时使用跳跃表和字典来保存有序集合对象，但这两种数据结构都会通过指针来共享相同元素的成员和分值，因此不会造成内存浪费。
### 为什么同时使用跳跃表和字典来实现？
字典查找成员效率快，跳跃表执行范围操作效率快，综合考虑两者结合起来使用。


## 类型:结构映射图
<img src="https://raw.githubusercontent.com/dark-tone/notes/main/Redis/imgs/1.png" width="736" height="222"/>
<br>
