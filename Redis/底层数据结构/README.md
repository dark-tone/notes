# SDS（简单动态字符串）
### 概念
simple dynamic string，Redis自身实现的抽象字符串类型，Redis的默认字符串表示均用的是SDS。所有键值对的键均为SDS底层实现。（C字符串只会作为字符串字面量来使用）

### 结构定义
**代码：**
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

<img src="http://www.gtnote.xyz/wp-content/uploads/2021/09/QQ截图20210909055648.jpg" alt="" width="532" height="220" class="alignnone size-full wp-image-41" />

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
### 结构定义
**代码：**
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
<img src="http://www.gtnote.xyz/wp-content/uploads/2021/09/QQ截图20210909074216.jpg" alt="" width="660" height="219" class="alignnone size-full wp-image-54" />


# 字典
### 结构定义
**代码:**
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
**示意图：**
<img src="http://www.gtnote.xyz/wp-content/uploads/2021/09/QQ截图20210910065428-1024x550.jpg" alt="" width="720" height="387" class="alignnone size-large wp-image-59" />

### 哈希算法
