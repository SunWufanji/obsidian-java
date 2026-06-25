# Redis 底层数据结构

## 1.1 SDS 动态字符串

Redis 没有直接使用 C 语言的字符串，而是自己实现了一种 **SDS（Simple Dynamic String）**。

```c
struct __attribute__ ((__packed__)) sdshdr8 {
    uint8_t  len;     // 已使用字节数
    uint8_t  alloc;   // 申请的总字节数
    uint8_t  flags;   // 头类型
    char     buf[];   // 字节数组
};
```

**SDS 的优势：**

| 特性 | C 字符串 | SDS |
|------|---------|-----|
| 获取长度 | O(n) 遍历 | O(1) 读 len |
| 缓冲区溢出 | 可能 | 自动扩容 |
| 内存分配 | 每次修改都分配 | 预分配 + 惰性释放 |
| 二进制安全 | 否（\0 截断） | 是 |

**动态扩容规则：**
- 新字符串长度 < 1MB：新空间 = 新长度 × 2
- 新字符串长度 ≥ 1MB：新空间 = 新长度 + 1MB

---

## 1.2 IntSet 整数集合

IntSet 是 Redis 中 Set 集合的一种实现，当 Set 中**所有元素都是整数**且**元素数量较少**时使用。

```c
typedef struct intset {
    uint32_t encoding;  // 编码方式：INT16 / INT32 / INT64
    uint32_t length;    // 元素个数
    int8_t   contents[]; // 整数数组（升序排列）
} intset;
```

**编码升级：** 当新元素超出当前编码范围时，自动升级（INT16 → INT32 → INT64），**只升不降**。

**查找：** 二分查找，O(log n)。

---

## 1.3 Dict 哈希表

Dict 是 Redis 中最核心的数据结构，用于存储所有 key-value 对。

```c
typedef struct dict {
    dictType *type;
    dictht    ht[2];   // 两个哈希表，用于 rehash
    long      rehashidx; // rehash 进度，-1 表示未在 rehash
    // ...
} dict;
```

**哈希冲突：** 链地址法（拉链法）。

**扩容触发条件：**
- LoadFactor（已用 / 总容量）≥ 1，且未在执行 bgsave / bgrewriteaof
- LoadFactor > 5（强制扩容，无论是否在持久化）

**收缩触发条件：** LoadFactor < 0.1

**扩/缩容大小：** 第一个大于等于 `used + 1`（扩容）或 `used`（收缩）的 2^n

**渐进式 rehash：**
- 同时维护 ht[0]（旧）和 ht[1]（新）
- 每次访问 Dict 时，执行一次 rehash（将 ht[0] 中一个 bucket 迁移到 ht[1]）
- rehash 期间：新增只写 ht[1]，查询/删除/修改在两个表中都操作
- rehash 完成后，ht[1] 替换 ht[0]

---

## 1.4 ZipList 压缩列表

ZipList 是一种**连续内存**的特殊"双端链表"，节省内存，但不适合存储大量数据。

**整体结构：**


| 字段      | 类型       | 大小  | 说明                 |
| ------- | -------- | --- | ------------------ |
| zlbytes | uint32_t | 4B  | 整个 ZipList 占用字节数   |
| zltail  | uint32_t | 4B  | 尾节点距起始地址的偏移量       |
| zllen   | uint16_t | 2B  | 节点数量（超过 65534 需遍历） |
| entry   | 不定       | 不定  | 各节点                |
| zlend   | uint8_t  | 1B  | 结束标志 0xFF          |

**Entry 结构：**
- `previous_entry_length`：前一节点长度（1 或 5 字节）
- `encoding`：数据类型和长度（1、2 或 5 字节）
- `contents`：实际数据

**连锁更新问题：** 当多个连续节点长度在 250~253 字节之间时，插入一个 ≥254 字节的节点，会导致后续节点的 `previous_entry_length` 从 1 字节扩展为 5 字节，引发连锁扩容。

**ZipList 特点：**
- 连续内存，内存占用低
- 节点间通过长度寻址，无需指针
- 数据过多时查询性能下降
- 增删大数据可能触发连锁更新

---

## 1.5 QuickList

QuickList 是 Redis 3.2 引入的数据结构，本质是**以 ZipList 为节点的双端链表**，兼顾内存效率和存储上限。

```
QuickList → [ZipList] ↔ [ZipList] ↔ [ZipList]
```

**配置项 `list-max-ziplist-size`：**
- 正数：每个 ZipList 最多存 N 个 entry
- 负数：限制每个 ZipList 的内存大小
  - -1：4KB，-2：8KB（默认），-3：16KB，-4：32KB，-5：64KB

**QuickList 特点：**
- 双端链表，节点为 ZipList
- 控制 ZipList 大小，解决连续内存申请效率问题
- 中间节点可压缩（LZF 算法），进一步节省内存

---

## 1.6 SkipList 跳表

SkipList 是一种**有序链表**，通过多层指针实现近似二分查找的效率。

**特点：**
- 元素按 score 升序排列
- 每个节点包含多层指针（层数 1~32，随机决定）
- 层级越高，跨度越大，查找时从高层开始跳跃
- 增删改查效率与红黑树相当（O(log n)），实现更简单

---

## 1.7 RedisObject

Redis 中所有 key 和 value 都被封装为 **RedisObject**：

```c
typedef struct redisObject {
    unsigned type:4;      // 数据类型（String/Hash/List/Set/ZSet）
    unsigned encoding:4;  // 编码方式
    unsigned lru:24;      // LRU 时钟（用于内存淘汰）
    int      refcount;    // 引用计数
    void    *ptr;         // 指向实际数据
} robj;
```

**11 种编码方式：**

| 编号 | 编码 | 说明 |
|------|------|------|
| 0 | OBJ_ENCODING_RAW | raw 动态字符串 |
| 1 | OBJ_ENCODING_INT | long 整数 |
| 2 | OBJ_ENCODING_HT | 哈希表 |
| 5 | OBJ_ENCODING_ZIPLIST | 压缩列表 |
| 6 | OBJ_ENCODING_INTSET | 整数集合 |
| 7 | OBJ_ENCODING_SKIPLIST | 跳表 |
| 8 | OBJ_ENCODING_EMBSTR | embstr 动态字符串 |
| 9 | OBJ_ENCODING_QUICKLIST | 快速列表 |
| 10 | OBJ_ENCODING_STREAM | Stream 流 |

---

## 1.8 五种数据类型的底层编码

### String
![[Pasted image 20260514210007.png|697]]

| 条件 | 编码 |
|------|------|
| 整数且在 LONG_MAX 范围内 | INT（直接存在 ptr 指针位置） |
| 字符串长度 ≤ 44 字节 | EMBSTR（RedisObject 与 SDS 连续内存） |
| 字符串长度 > 44 字节 | RAW（SDS 单独分配） |

### List
![[Pasted image 20260514210305.png|639]]

| 版本 | 条件 | 编码 |
|------|------|------|
| 3.2 以前 | 元素 ≤ 512 且每个元素 ≤ 64B | ZipList |
| 3.2 以前 | 超出上述条件 | LinkedList |
| 3.2 以后 | 统一 | QuickList |

### Hash

| 条件 | 编码 |
|------|------|
| 元素数量 ≤ `hash-max-ziplist-entries`（默认 512）且每个元素 ≤ `hash-max-ziplist-value`（默认 64B） | ZipList |
| 超出上述条件 | HT（Dict） |

ZipList 中相邻两个 entry 分别保存 field 和 value。

### Set

| 条件 | 编码 |
|------|------|
| 所有元素都是整数且数量 ≤ `set-max-intset-entries` | IntSet |
| 其他情况 | HT（Dict，value 统一为 null） |

### SortedSet（ZSet）
![[Pasted image 20260514212540.png]]

| 条件 | 编码 |
|------|------|
| 元素数量 ≤ `zset-max-ziplist-entries`（默认 128）且每个元素 ≤ `zset-max-ziplist-value`（默认 64B） | ZipList |
| 超出上述条件 | SkipList + HT（同时维护，SkipList 负责排序，HT 负责 O(1) 查分数） |
![[Pasted image 20260514212457.png]]
ZipList 中 element 和 score 紧挨存储，score 越小越靠近队首。

---

## 面试简答

**Q: Redis 的 SDS 相比 C 字符串有什么优势？**

A: 三点：O(1) 获取长度（记录了 len）；自动扩容不会溢出；二进制安全（不以 \0 判断结尾）。

**Q: Dict 的 rehash 为什么是渐进式的？**

A: 如果一次性迁移所有数据，数据量大时会阻塞 Redis 很长时间。渐进式 rehash 将迁移分散到每次访问中，每次只迁移一个 bucket，避免长时间阻塞。

**Q: ZipList 的连锁更新是什么？**

A: ZipList 每个节点用 `previous_entry_length` 记录前一节点长度，占 1 或 5 字节。当多个连续节点长度在 250~253 字节时，插入一个大节点会导致后续节点都需要将 1 字节扩展为 5 字节，引发连锁的内存重新分配。

**Q: ZSet 为什么同时用 SkipList 和 HT？**

A: SkipList 支持按 score 排序和范围查询，但按 member 查 score 是 O(log n)；HT 支持 O(1) 按 member 查 score。两者结合，既能高效排序，又能 O(1) 查分数。
