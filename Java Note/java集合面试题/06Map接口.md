# Map 接口

## 1. 一个简单的 HashMap 实际例子

### 先讲场景

HashMap 最常用的场景就两类：**"根据什么找什么"** 和 **"统计次数"**。

下面用两个最贴近开发实际的例子，让你一眼看懂 HashMap 怎么用。

---

### 例子 1：根据 key 找 value（字典查询）

```java
// 场景：用学号查学生名字
Map<String, String> studentMap = new HashMap<>();

// 存数据（put）
studentMap.put("001", "张三");
studentMap.put("002", "李四");
studentMap.put("003", "王五");

// 查数据（get）
String name = studentMap.get("002");
System.out.println(name);  // 李四

// 查不存在的 key 返回 null
String name2 = studentMap.get("999");
System.out.println(name2);  // null

// 判断 key 是否存在
if (studentMap.containsKey("001")) {
    System.out.println("001 存在");  // ✅ 输出
}
```

**怎么理解？** 就像你有一个**通讯录**，key 是姓名，value 是电话号码——输入 key，直接拿到 value。

---

### 例子 2：统计次数（高频场景）

```java
// 场景：统计一段话里每个单词出现了几次
String text = "a b c a b a";
String[] words = text.split(" ");

Map<String, Integer> countMap = new HashMap<>();

for (String word : words) {
    // 关键写法：getOrDefault
    countMap.put(word, countMap.getOrDefault(word, 0) + 1);
}

System.out.println(countMap);
// 输出：{a=3, b=2, c=1}
```

**为什么用 `getOrDefault`？**

```java
// 第一次遇到 "a" 时：countMap 里还没有 "a"
// getOrDefault("a", 0) → 返回 0
// put("a", 0 + 1)     → "a" = 1

// 第二次遇到 "a" 时：countMap 里有 "a" = 1
// getOrDefault("a", 0) → 返回 1
// put("a", 1 + 1)     → "a" = 2
```

如果不用 `getOrDefault`，你得自己先判断 key 存不存在：

```java
// 等价写法（更啰嗦）
for (String word : words) {
    if (countMap.containsKey(word)) {
        countMap.put(word, countMap.get(word) + 1);
    } else {
        countMap.put(word, 1);
    }
}
```

---

### 例子 3：用自定义对象做 key（⚠️ 注意）

```java
// 正确做法：重写 hashCode + equals
class Student {
    String id;
    String name;
    
    Student(String id, String name) {
        this.id = id;
        this.name = name;
    }
    
    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o instanceof Student s) {
            return Objects.equals(this.id, s.id);
        }
        return false;
    }
    
    @Override
    public int hashCode() {
        return Objects.hash(id);  // 用 id 算 hashCode
    }
}

Map<Student, Integer> scoreMap = new HashMap<>();
scoreMap.put(new Student("001", "张三"), 90);
scoreMap.put(new Student("001", "张三"), 95);  // ✅ 同一个 key → 覆盖

System.out.println(scoreMap.size());  // 1
System.out.println(scoreMap.get(new Student("001", "张三")));  // 95
```

和 HashSet 一样的道理：**自定义对象做 key，必须重写 hashCode + equals**。

---

### 开发中最常用的 HashMap 方法

```java
Map<String, Integer> map = new HashMap<>();

// 增
map.put("A", 1);              // 放一个键值对

// 删
map.remove("A");              // 删掉 key 为 "A" 的条目

// 改
map.put("A", 2);              // key 已存在 → 覆盖旧值

// 查
map.get("A");                 // 拿 value，没有返回 null
map.getOrDefault("A", 0);     // 拿 value，没有返回默认值 0
map.containsKey("A");         // 判断 key 是否存在
map.containsValue(1);         // 判断 value 是否存在（少用，性能差）

// 遍历
for (Map.Entry<String, Integer> entry : map.entrySet()) {
    System.out.println(entry.getKey() + " = " + entry.getValue());
}

// 其他
map.size();                   // 有多少个键值对
map.isEmpty();                // 是不是空的
map.clear();                  // 清空所有
```

### 一句话记住

> HashMap = 通讯录：**输入 key 秒查 value**，统计次数是经典用法

### 面试简答版

- 最常用法：`put(key, value)` 存，`get(key)` 取
- 统计次数经典写法：`map.put(key, map.getOrDefault(key, 0) + 1)`
- 自定义对象做 key 必须重写 `hashCode` + `equals`
- 判断 key 存在用 `containsKey()`，遍历用 `entrySet()`

## 2. 什么是 Hash 算法？

### 先讲场景

HashMap 之所以能"根据 key 秒查 value"，核心秘密就是 **Hash 算法**。

但"Hash"这个词听起来很抽象，到底它干了什么事？

```java
map.put("张三", 90);
// "张三"这个 key 是怎么决定自己该放在数组的哪个位置的？
```

**一句话：Hash 算法就是把任意大小的数据，压缩成一个固定长度的数字（hashCode）。**

---

### 用个简单的例子理解

假设你有一排 16 个抽屉（数组），要给每个人分配一个抽屉放东西。

你怎么决定"张三"放几号抽屉？

```java
// 一个超级简单的 Hash 算法：
// 把名字的每个字的拼音首字母转成数字，再对 16 取余

"张三" → Z + S → 26 + 19 = 45 → 45 % 16 = 13

// 结论：张三放 13 号抽屉
```

这就是 Hash 算法的核心工作：**把"张三"这个字符串，算成一个 0~15 之间的数字（数组下标）。**

---

### 真实的 hashCode 是怎么算的？

```java
// String 的 hashCode 计算方法（JDK）
// 公式：s[0]*31^(n-1) + s[1]*31^(n-2) + ... + s[n-1]

String name = "张三";
System.out.println(name.hashCode());  // 比如 774573

// 拿到 hashCode 后，HashMap 还要处理一下才能变成数组下标
int hash = name.hashCode();           // 774573
int index = (hash ^ (hash >>> 16)) & (16 - 1);  // 最终下标
```

`hashCode` 是一个很大的 int 值（可能是几十亿），HashMap 不能直接用这么大的数当数组下标，需要通过**二次计算**把它映射到数组长度范围内。

---

### Hash 算法的三个核心要求

| 要求 | 什么意思 | 如果不满足会怎样 |
|:---|:---|:---|
| **确定性** | 同一个输入，每次算出来结果一样 | HashMap 存的 key 就找不到了 |
| **速度快** | 计算本身不能太耗时 | 每次 put/get 都很慢 |
| **分布均匀** | 不同输入尽量散开，不要挤在一起 | 哈希冲突多 → 链表长 → 性能退化成 O(n) |

第三点最要命。好的 Hash 算法让元素均匀分布：

```
❌ 差的 Hash：全部挤在一起            ✅ 好的 Hash：均匀散开
┌──────────────┐                    ┌──────────────┐
│ 0: A → B → C │                    │ 0: A          │
│ 1:            │                    │ 1: B          │
│ 2:            │                    │ 2:            │
│ 3: D → E → F │                    │ 3: C          │
│ ...           │                    │ 4: D          │
└──────────────┘                    └──────────────┘
    查找退化成 O(n)                      查找 O(1)
```

---

### Java 里最经典的 Hash 算法：String.hashCode()

```java
// 公式：s[0]*31^(n-1) + s[1]*31^(n-2) + ... + s[n-1]
// 为什么选 31？因为它是一个奇素数，而且 31 * i = (i << 5) - i，JVM 能优化成位运算

"ABC".hashCode() = 'A' * 31² + 'B' * 31 + 'C'
                 = 65 * 961 + 66 * 31 + 67
                 = 62465 + 2046 + 67
                 = 64578
```

---

### Hash 冲突

Hash 算法虽然好，但没法避免两个不同的输入算出同一个结果：

```java
// 经典例子
"Aa".hashCode();   // 2112
"BB".hashCode();   // 2112 ← 和 "Aa" 一样！
```

这就是**哈希冲突**。HashMap 用**拉链法**解决：同一个位置挂一个链表（或红黑树）。

```java
// 数组下标 5 挂了两个 key
index 5: "Aa" → "BB"
// 查找时先定位到下标 5，再遍历链表用 equals 找到对的 key
```

---

### 总结

```
Hash 算法 = 把任意数据压缩成固定数字（hashCode）
HashMap 用它来：决定 key 放在数组的哪个位置
好 Hash 的标准：快、均匀、确定
解决冲突：拉链法（链表/红黑树）
```

### 面试回答

> **面试官：什么是 Hash 算法？**
>
> Hash 算法就是把任意大小的数据，计算成一个固定长度的数字（hashCode）。HashMap 用这个 hashCode 来决定 key 放在数组的哪个位置。
>
> 好的 Hash 算法要做到三点：计算快、分布均匀、同样的输入得到同样的结果。Java 里 String 的 hashCode 就是典型，用 31 作为乘数，兼顾了速度和均匀性。
>
> 无论多好的算法都可能有哈希冲突（不同的 key 算出相同的 hashCode），HashMap 用链表和红黑树来解决。

### 面试简答版

- Hash算法 = 把任意数据转成固定数字（hashCode）
- HashMap用它把 key 映射到数组下标
- 三个要求：快、均匀、确定
- 哈希冲突无法完全避免，HashMap 用链表/红黑树解决（拉链法）

## 3. 什么是链表？

### 先讲场景

数组大家都很熟——一排连续的内存格子，有索引，`get(3)` 直接拿第 4 个。

但数组有个毛病：**中间插入/删除很慢**（要移位）。于是就有了另一种结构——链表。

**链表是一串节点，每个节点里存着"数据"和"指向下一个节点的指针"，像锁链一样连起来。**

---

### 数组 vs 链表

```
数组（连续内存）：               链表（分散内存，靠指针连）：
┌────┬────┬────┬────┬────┐     ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐
│ A  │ B  │ C  │ D  │ E  │     │ A  │──→│ B  │──→│ C  │──→│ D  │──→null
└────┴────┴────┴────┴────┘     └──────┘  └──────┘  └──────┘  └──────┘
  0    1    2    3    4          地址:100  地址:205  地址:310  地址:450
```

**数组**：大家紧挨着坐，你坐在 3 号位，旁边就是 4 号位。
**链表**：大家散坐在不同地方，但每个人都指着下一个人在哪。

```java
// 链表的节点长这样
class Node {
    String data;    // 存的数据
    Node next;      // 指向下一个节点的指针
    
    Node(String data) {
        this.data = data;
        this.next = null;
    }
}
```

---

### 实际写一个最简单的链表

```java
// 手写一个最简单的单向链表
class Node {
    String value;
    Node next;
    
    Node(String value) { this.value = value; }
}

// 创建链表：A → B → C → null
Node head = new Node("A");
Node nodeB = new Node("B");
Node nodeC = new Node("C");

head.next = nodeB;    // A 指向 B
nodeB.next = nodeC;   // B 指向 C

// 遍历链表：从 head 开始，一个一个往后走
Node current = head;
while (current != null) {
    System.out.println(current.value);
    current = current.next;  // 走到下一个
}
// 输出：A B C
```

---

### 链表和数组的区别

| 对比 | 数组 | 链表 |
|:---|:---|:---|
| **内存** | 连续的一块 | 分散的，靠指针连 |
| **按索引查** | **O(1)** ✅ 直接算地址 | **O(n)** ❌ 要一个个走 |
| **头部插入** | **O(n)** ❌ 要移位 | **O(1)** ✅ 改头指针就行 |
| **中间插入** | **O(n)** ❌ 要移位 | **O(n)** ❌ 要先找到位置 |
| **尾部插入** | O(1) 均摊 ✅ | O(1) 有尾指针 ✅ |
| **多占内存** | 只存数据 ✅ | 每个节点多存指针 ❌ |

---

### 链表的两种形式

#### 单向链表

```
A → B → C → D → null
```

每个节点只知道下一个是谁，不知道上一个是谁。只能从头往后走。

#### 双向链表（Java LinkedList 用的就是这种）

```
null ← A → B → C → D → null
       ←   ←   ←
```

每个节点有 `prev` 和 `next` 两个指针，既能往前走也能往后走。

```java
// 双向链表的节点
class DoublyNode {
    String value;
    DoublyNode prev;  // 指向前一个
    DoublyNode next;  // 指向后一个
}
```

---

### 链表在 Java 集合里的身影

```
1. LinkedList       → 本身就是双向链表
2. HashMap 的拉链法  → 数组 + 链表（解决哈希冲突）
3. LinkedHashMap    → HashMap + 双向链表（保持顺序）
```

### 面试回答

> **面试官：什么是链表？**
>
> 链表是一种数据结构，由一个个节点串起来。每个节点存着数据和指向下一个节点的指针，内存不连续，靠指针连接。
>
> 和数组比：链表插入删除快（O(1)，改指针就行），但查找慢（O(n)，要遍历）。数组查找快（O(1)），但插入删除慢（O(n)，要移位）。
>
> Java 里 LinkedList 就是双向链表，HashMap 也用链表来解决哈希冲突。

### 面试简答版

- 链表 = 节点串起来，靠指针连接，内存不连续
- 插入/删除快（改指针），查找慢（要遍历）
- 数组查找快（有索引），插入/删除慢（要移位）
- Java 里：LinkedList（双向链表）、HashMap（拉链法用链表）

## 4. 说一下 HashMap 的实现原理？

### 先讲场景

你已经知道 HashMap 能"根据 key 秒查 value"，但它底层到底是怎么做到的呢？put 一个键值对进去，它经历了什么？为什么同样的代码在 JDK 1.7 和 1.8 上行为不一样？

这些问题就是面试问"HashMap 实现原理"时想听的。

**核心一句话：HashMap 底层是"数组 + 链表 + 红黑树"（JDK 1.8+），put 时算 hash 定位数组下标，冲突了就用链表或红黑树存。**

---

### 底层数据结构

```
JDK 1.8+ 的 HashMap：
┌────────────────────────────────────────────────────────────┐
│  table（Node 数组）                                        │
│  ┌────┬────┬────┬────┬────┬────┬────┬────┬────┬────┐     │
│  │  0 │  1 │  2 │  3 │  4 │  5 │  6 │  7 │  8 │  9 │     │
│  ├────┼────┼────┼────┼────┼────┼────┼────┼────┼────┤     │
│  │    │ K1 │    │ K3 │    │K5→K6│   │    │ K8 │    │     │
│  │    │    │    │ ↓  │    │ ↓   │   │    │ ↓  │    │     │
│  │    │ K2 │    │ K4 │    │ K7  │   │    │红黑 │    │     │
│  │    │    │    │    │    │     │   │    │树   │    │     │
│  └────┴────┴────┴────┴────┴────┴────┴────┴────┴────┘     │
│                                                            │
│  每个格子可能的情况：                                       │
│  - null          → 还没放过东西                             │
│  单个 Node       → 没冲突，直接放                           │
│  链表（≤7 个）   → 冲突了，链起来                           │
│  红黑树（≥8 个） → 链表太长，转成树提高查询性能               │
└────────────────────────────────────────────────────────────┘
```

每个 Node 内部：

```java
static class Node<K,V> {
    final int hash;
    final K key;
    V value;
    Node<K,V> next;
}
```

---

### put 方法核心流程

```java
map.put("张三", 90);
```

JVM 里实际干了五件事：

```
① 算 hash：key.hashCode() → 再扰动 (h ^ (h >>> 16))
② 定位：hash & (table.length - 1) → 得到数组下标
③ 如果没人：直接 new Node 放进去
④ 如果有人：equals 比较 key
   ├── key 相同 → 覆盖 value
   └── key 不同 → 挂在链表/红黑树上
⑤ 如果元素个数 > 阈值（容量 × 0.75）→ 扩容
```

画成流程图更清楚：

```
put(key, value)
    │
    ├─→ key == null ? → 放在 table[0]（HashMap 允许 null key）
    │
    └─→ 计算 hash = (h = key.hashCode()) ^ (h >>> 16)
         │
         └─→ index = hash & (tab.length - 1)
              │
              ├─→ table[index] == null → 直接放进去 ✅
              │
              └─→ table[index] != null → 哈希冲突了
                   │
                   ├─→ key 相同（equals 为 true）→ 覆盖 value
                   │
                   └─→ key 不同
                        ├─→ 链表节点数 < 8 → 尾插法挂到链表
                        └─→ 链表节点数 ≥ 8 → 转成红黑树再插入
```

---

### get 方法核心流程

```java
map.get("张三");
```

```
get(key)
    │
    └─→ 算 hash（和 put 一样的方法）
         │
         └─→ index = hash & (length - 1)
              │
              ├─→ table[index] == null → return null
              │
              └─→ table[index] != null
                   ├─→ 先检查第一个节点：key.equals(key) → 是就返回 value
                   └─→ 不是 → 遍历链表/红黑树找 equals 为 true 的那个
```

---

### 为什么要用 `hash & (length - 1)` 而不是 `hash % length`？

因为数组长度始终是 **2 的幂**，`hash & (length - 1)` 等价于 `hash % length`，但**位运算比取模快得多**。

```java
// 数组长度 16（二进制 10000）
// length - 1 = 15（二进制 01111）
// hash & 15  = 只保留 hash 的低 4 位 → 结果范围 0~15
// 相当于 hash % 16，但 & 运算只要一个 CPU 周期
```

这也是为什么 HashMap 的容量**必须**是 2 的幂——如果自己传一个不是 2 的幂的初始容量，HashMap 也会帮你调整成最接近的 2 的幂。

---

### 扩容机制

```java
// 默认值
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4;   // 16
static final float DEFAULT_LOAD_FACTOR = 0.75f;

// 扩容触发条件
// size（元素个数）> capacity × loadFactor
// 默认：size > 16 × 0.75 = 12 → 扩容为 32
```

扩容时元素要么**留在原位**，要么**移到"原位置 + 旧容量"的位置**：

```
旧数组（长度 16）：               新数组（长度 32）：
┌────┬────┬────┬────┐            ┌────┬────┬────┬────┬────┬────┐
│    │ K1 │    │ K3 │  扩容→    │    │    │ K1 │    │ K3 │    │
│    │    │    │    │            │    │    │    │    │    │    │
└────┴────┴────┴────┘            └────┴────┴────┴────┴────┴────┘
 旧下标 1   旧下标 3              新下标 1   新下标 3
                                 或 17      或 19
```

判断依据：

```java
// 旧容量 16 → 看 hash 的第 5 位（从右数）是 0 还是 1
// hash 第 5 位 = 0 → 留在原位
// hash 第 5 位 = 1 → 移到 原位置 + 16
```

这样设计的好处：**不需要重新计算每个 key 的 hash，只用看新增的那一位是 0 还是 1**，效率极高。

---

### JDK 1.7 vs 1.8 关键区别

| 对比项        | JDK 1.7           | JDK 1.8             |
| :--------- | :---------------- | :------------------ |
| 数据结构       | 数组 + 链表           | 数组 + 链表 + **红黑树**   |
| 插入方式       | **头插法**           | **尾插法**             |
| 扩容后 rehash | 重新计算每个 key 的 hash | 位运算判断原位/挪位          |
| 链表 → 红黑树   | 无                 | 链表长度 ≥ 8 且数组长度 ≥ 64 |
| 红黑树 → 链表   | 无                 | 红黑树节点 ≤ 6           |

**为什么头插法改尾插法？**

头插法在并发 put 时会产生**环形链表**，导致 get 时死循环。虽然 HashMap 本身就不保证线程安全，但这个 bug 太出名了，JDK 1.8 改成尾插法来规避。

---

### 几个容易忽略的关键点

**1. 链表转红黑树的条件**

不是链表长度 ≥ 8 就一定转树，还要满足 **数组长度 ≥ 64**。如果数组还比较小（< 64），会优先扩容，而不是转树。

```java
// 为什么？
// 数组小的时候，冲突可能是因为容量不够，扩容后元素散开，链表自然就短了
// 数组已经 ≥ 64 了还冲突，说明这些 key 的 hashCode 本身就分布不好，才用树来兜底
```

**2. 负载因子 0.75 为什么是默认值**

空间和时间的折中：
- **太大（比如 1）**：空间利用率高，但冲突多，链表长，查询慢
- **太小（比如 0.5）**：冲突少，查询快，但太浪费空间，频繁扩容也影响性能
- **0.75**：经过大量测试，是空间和时间的平衡点

**3. null key 的处理**

HashMap 允许 null 作为 key，它被特殊处理，**固定放在 table[0]**：

```java
// HashMap.put 方法开头
if (key == null)
    return putForNullKey(value);  // 专门处理 null key
```

---

### 面试回答

> **面试官：说一下 HashMap 的实现原理？**
>
> HashMap 底层是数组 + 链表 + 红黑树（JDK 1.8+）。
>
> put 时，先算 key 的 hashCode，再经过一次扰动（hash ^ hash >>> 16）降低碰撞概率，然后用 `(n - 1) & hash` 定位到数组下标。如果那个位置空的，直接放进去；如果有元素，用 equals 比较 key，相同就覆盖 value，不同就挂到链表或红黑树后面。
>
> get 时流程类似，算 hash → 定位下标 → 用 equals 找 key。
>
> 扩容触发条件是元素个数超过容量 × 0.75（默认是 16 × 0.75 = 12），每次扩容成原来的两倍。扩容后元素要么留在原位，要么移到"原位置 + 旧容量"的位置，靠的是看 hash 值新增的那一位是 0 还是 1。
>
> JDK 1.8 相比 1.7 最大的变化：引入了红黑树优化长链表的查询性能、尾插法代替头插法避免了并发死循环的隐患、扩容时不再重新计算 hash。

### 面试简答版

- 底层结构：**数组 + 链表 + 红黑树**（JDK 1.8+）
- put 流程：算 hash → 定位下标 → 空则直接放，非空则 equals 比较 → 相同覆盖，不同链/树
- get 流程：算 hash → 定位下标 → equals 找 key
- 扩容：**size > 容量 × 0.75** 时触发，每次扩为 2 倍，元素原位或挪"原位置 + 旧容量"
- JDK 1.7 → 1.8：头插→尾插、加红黑树、rehash 优化
- 关键点：容量必须是 2 的幂、null key 放 table[0]、链表≥8 且数组≥64 才转树

## 5. HashMap 在 JDK 1.7 和 1.8 中有哪些不同？还有哪些集合有类似更新？

### 先讲场景

上一题说了 HashMap 的实现原理，里面提到了 JDK 1.8 引入了红黑树、头插改尾插。但面试经常单独拎出来问："**JDK 1.7 和 1.8 的 HashMap 具体有什么不同？为什么这么改？**"

而且还会追问："**还有哪个集合也经历了类似的升级？**"——答案是 `ConcurrentHashMap`，升级路线几乎一模一样。

---

### 区别一：底层数据结构

```
JDK 1.7：                         JDK 1.8：
┌────┬────┬────┬────┐             ┌────┬────┬────┬────┬────┐
│ 0  │ 1→2│ 3  │ 4  │             │ 0  │ 1→2│ 3  │ 4→5→6│ 7  │
└────┴────┴────┴────┘             │    │    │ ↓  │ ↓    │    │
  数组 + 链表（只有链表）             │    │    │红黑 │红黑   │    │
                                    │    │    │树   │树     │    │
                                    └────┴────┴────┴───────┴────┘
                                      数组 + 链表 + 红黑树
```

JDK 1.7 只有链表，最坏情况下冲突全堆在一个桶里，查找退化成 **O(n)**。
JDK 1.8 加了红黑树，链表 ≥ 8 且数组 ≥ 64 时转树，查找变成 **O(log n)**。

---

### 区别二：插入方式（头插法 vs 尾插法）——面试重点

```java
// JDK 1.7：头插法
void addEntry(int hash, K key, V value, int bucketIndex) {
    Entry<K,V> e = table[bucketIndex];
    table[bucketIndex] = new Entry<>(hash, key, value, e);
    // 新节点.next = 旧头 → 自己变成新头
}
```

```
头插法效果：插入顺序 A → B → C
实际链表：C(head) → B → A → null（后插入的反而在前面）
```

```java
// JDK 1.8：尾插法
// 遍历到链表尾部，然后 p.next = newNode(...)
```

```
尾插法效果：插入顺序 A → B → C
实际链表：A(head) → B → C → null（按插入顺序排列）
```

#### 为什么改？头插法在并发扩容时会死循环

要理解为什么头插法会导致环形链表，得先看 JDK 1.7 的扩容核心方法 `transfer()`：

```java
// JDK 1.7 HashMap.transfer() 核心逻辑
void transfer(Entry[] newTable) {
    Entry[] src = table;
    int newCapacity = newTable.length;
    
    for (int j = 0; j < src.length; j++) {
        Entry<K,V> e = src[j];          // e = 当前链表的头节点
        if (e != null) {
            src[j] = null;
            do {
                Entry<K,V> next = e.next;   // ① 先记下下一个节点
                int i = e.hash & (newCapacity - 1); // 重新算下标
                e.next = newTable[i];   // ② e.next 指向新数组该位置的旧头（头插）
                newTable[i] = e;        // ③ e 成为新头
                e = next;               // ④ 处理下一个节点
            } while (e != null);
        }
    }
}
```

这段代码配合头插法，**正常单线程**下是这样工作的：

```
初始链表放在旧数组位置：A → B → null

开始 rehash（假设 A、B 都映射到新数组的同一个位置）：

第 1 轮：e = A, next = B
  e.next = newTable[i] → A.next = null（null 是旧头）
  newTable[i] = A      → 新位置：A → null
  e = B

第 2 轮：e = B, next = null
  e.next = newTable[i] → B.next = A（A 是当前头）
  newTable[i] = B      → 新位置：B → A → null
  e = null → 结束

结果：B → A → null（顺序反转了 ✅）
```

##### 并发时到底怎么形成环形链表的？

两个线程同时触发扩容，用**分步图**来看这个过程：

```
初始状态：
旧数组某个桶：A → B → null
线程 1 和线程 2 都要把这个链表 rehash 到新数组
```

**第 1 步：线程 1 执行到一半被挂起**

```
线程 1 执行完 transfer 中的前两行：
  e = A         ← 当前要处理的节点
  next = B      ← 记下 A.next 备用

然后线程 1 的时间片到了，停在这里
```

**第 2 步：线程 2 完整执行完整个扩容**

```
线程 2 独立完成了全部 rehash，结果：

新数组该位置：B → A → null

注意看关键变量在新数组中的状态：
  A.next = null（A 变成了尾部）
  B.next = A（B 指向 A）
  
但线程 1 手头还拿着旧数据：
  e = A（旧引用）
  next = B（也还是旧引用）
```

**第 3 步：线程 1 恢复执行——灾难开始**

```
线程 1 从停下的地方继续执行 transfer 循环：
  
当前状态：
  e = A, next = B
  新数组已经有：B → A → null

第 1 轮循环（处理 e = A）：
  int i = A.hash & (newCapacity - 1)  → 得到下标
  A.next = newTable[i]                → A.next = B（！！拿新数组的头——B）
  newTable[i] = A                     → 新链表：A → B → A → ...
  e = next = B                        → e = B，准备下一轮
```

等等，这里需要更准确一些。让我重新梳理。

实际上在 JDK 1.7 的 transfer 中，当线程 2 已经完成了完整的 rehash 后，新数组对应位置已经有 `B → A → null`。

线程 1 恢复时：
- `e = A`（它之前拿到的旧引用）
- `next = B`（它之前拿到的 A.next，这个引用指向的是"旧 A 的 next"——但在并发下，这个引用其实指向的是对象 B，而 B 经过线程 2 的处理后，B.next 已经变成了 A）

所以：

```
第 1 轮循环（处理 e = A）：
  int i = A.hash & (newCapacity - 1)  // 算下标
  A.next = newTable[i]                // newTable[i] 是 B，所以 A.next = B
  newTable[i] = A                     // 新数组该位置：A → B → A ...
  e = next = B                        // e = B，继续

第 2 轮循环（处理 e = B）：
  next = B.next                       // B.next 是 A（线程 2 设的）
  B.next = newTable[i]               // newTable[i] = A，所以 B.next = A
  newTable[i] = B                     // 新数组该位置：B → A → B → A ...
  e = next = A                        // e = A，还要继续！！

第 3 轮循环（处理 e = A）：
  next = A.next                       // A.next 是 B（上轮设的）
  A.next = newTable[i]               // newTable[i] = B，所以 A.next = B
  newTable[i] = A                     // 又回到 A → B → A ...
  e = next = B                        // e = B，又来了！！！
```

**这是一个死循环，永远不会结束。** 因为 A 和 B 互相指着对方，遍历链表时永远走不到 null。

```
最终结果（环形链表）：
  A.next = B
  B.next = A
  
结构：
  → A → B → A → B → A → ...（死循环 🔁）
```

这时候如果有线程来 get 这个桶上的元素，会在遍历时永远出不来，CPU 100%。

##### 图解整个过程

```
① 初始状态：
   旧链表：A → B → null

② 线程 1 挂起，线程 2 完成扩容：
   新链表：B → A → null（正常 rehash）
   线程 1 手头抓着 e = A, next = B

③ 线程 1 恢复：
   ┌─────────────────────────────────────┐
   │ 处理 A：A.next = newTable[i] (= B)  │
   │        A 插到头：位置 → A → B       │
   │        此时 B.next 还指向 A ！       │
   ├─────────────────────────────────────┤
   │ 处理 B：B.next = newTable[i] (= A)  │
   │        B 插到头：位置 → B → A → B   │
   ├─────────────────────────────────────┤
   │ 处理 A（又来）：A.next = B           │
   │ 处理 B（又再来）：B.next = A         │
   │ ... 永无止境 🔁                      │
   └─────────────────────────────────────┘
```

JDK 1.8 改尾插法，扩容时不会反转链表顺序，**从根源上避免了环形链表**。但这并不代表 HashMap 是线程安全的——并发下还是用 `ConcurrentHashMap`。

---

### 区别三：扩容时的 rehash 方式

```java
// JDK 1.7：重新算每个 key 的下标
int newIndex = e.hash & (newCapacity - 1);
// 每个 key 都要执行一次 & 运算

// JDK 1.8：不用算，看 hash 的新增那一位
// 旧容量 16（二进制 10000）→ 新容量 32（二进制 100000）
// 判断：hash & 10000
//   = 0 → 留在原位
//   = 1 → 移到 原位 + 16
```

JDK 1.7 每个 key 都要重新 hash 一次，JDK 1.8 只用看新增的 1 位是 0 还是 1，**不需要重新计算**。

---

### 区别四：hash 扰动函数

```java
// JDK 1.7：4 次位运算
static int hash(int h) {
    h ^= (h >>> 20) ^ (h >>> 12);
    return h ^ (h >>> 7) ^ (h >>> 4);
}

// JDK 1.8：1 次位运算
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

JDK 1.8 简化了扰动。为什么敢简化？因为引入了红黑树，即使冲突多也有树兜底，不需要在 hash 阶段做得太复杂。

---

### 全部区别一张表

| 对比项 | JDK 1.7 | JDK 1.8 |
|:---|:---|:---|
| **数据结构** | 数组 + 链表 | 数组 + 链表 + **红黑树** |
| **插入方式** | **头插法**（新节点插链表头） | **尾插法**（新节点挂链表尾） |
| **扩容 rehash** | 重新算每个 key 的 index | 位运算判断原位/原位+旧容量 |
| **哈希扰动** | 4 次位运算 | 1 次位运算 |
| **链表→红黑树** | 无 | ≥ 8 且数组 ≥ 64 |
| **红黑树→链表** | 无 | ≤ 6 |
| **并发安全性** | 头插法可能死循环 | 尾插法避免死循环（但仍非线程安全） |

---

### 哪个集合也有类似的更新？

**ConcurrentHashMap 从 JDK 7 到 JDK 8 的升级，和 HashMap 几乎一样：**

| 对比项        | ConcurrentHashMap JDK 7          | ConcurrentHashMap JDK 8        |
| :--------- | :------------------------------- | :----------------------------- |
| **数据结构**   | Segment 数组 + HashEntry 数组 + 链表   | 数组 + 链表 + **红黑树**              |
| **锁机制**    | **分段锁**（ReentrantLock 锁 Segment） | **CAS + synchronized**（锁链表头节点） |
| **并发粒度**   | 默认 16 个 Segment                  | 每个数组元素一个锁                      |
| **链表→红黑树** | 无                                | ≥ 8 转树且数组>=64                  |

**升级思路完全一致：**
- 数据结构上：都引入了红黑树来解决长链表查询退化问题
- 锁粒度上：都变得更细——ConcurrentHashMap 从锁 Segment 变成了锁单个节点
- 性能上：都是大幅提升

详细内容可以看 [[03集合容器概述.md#8 哪些集合类是线程安全的？]] 里的 ConcurrentHashMap 深度解析。

---

### 面试回答

> **面试官：HashMap 在 JDK 1.7 和 1.8 有哪些区别？**
>
> 主要有五点不同：
>
> 第一，数据结构上，1.8 引入了红黑树，当链表长度 ≥ 8 且数组长度 ≥ 64 时转树，把最坏情况下的查找从 O(n) 降到 O(log n)。
>
> 第二，插入方式上，1.7 是头插法，1.8 改成了尾插法。头插法在并发扩容时会产生环形链表导致死循环，尾插法从根源上规避了这个问题。
>
> 第三，扩容 rehash 上，1.7 要重新计算每个 key 的 index，1.8 直接用位运算判断 hash 的新增位是 0 还是 1，决定元素留在原位还是移到"原位 + 旧容量"。
>
> 第四，hash 扰动函数从 1.7 的 4 次位运算简化为 1 次，因为红黑树兜底了，不需要在 hash 阶段做太复杂。
>
> 第五，ConcurrentHashMap 的升级路线和 HashMap 几乎一样——JDK 7 是 Segment 分段锁 + 链表，JDK 8 也改成了 CAS + synchronized + 红黑树。

### 面试简答版

- **数据结构**：1.7 数组+链表，1.8 加了**红黑树**
- **插入方式**：1.7 **头插法**（并发可能死循环），1.8 **尾插法**
- **扩容 rehash**：1.7 重新算，1.8 位运算判断原位/挪位
- **hash 扰动**：1.7 四次扰动，1.8 一次扰动
- **类似更新**：**ConcurrentHashMap** 从 JDK 7 的 Segment + ReentrantLock 升级为 JDK 8 的 CAS + synchronized + 红黑树

## 6. hash 值、hashCode、key 有什么区别？

### 先讲场景

看 HashMap 源码时，你会发现几个容易搞混的概念：

```java
// 这几个东西到底分别是什么？
map.put(key, value);         // key 是你传的对象
key.hashCode();              // 这是 key 自己算出来的 int
int hash = hash(key);        // 这是 HashMap 对 hashCode 又加工了一次
int index = hash & (n - 1); // 这才是最终的数组下标
```

很多人以为 `hashCode` = `hash` 值，其实不对——**hashCode 经过 HashMap 的"扰动"之后才叫 hash 值**。

搞清楚这三者的关系，才能真正理解 HashMap 的 put/get 寻址过程。

---

### 一句话区分

| 概念           | 本质                  | 谁产生的                    | 干什么用             |
| :----------- | :------------------ | :---------------------- | :--------------- |
| **key**      | 你传的对象               | 调用者                     | 存/取数据的依据         |
| **hashCode** | key 算出来的 int        | key 自己的 `hashCode()` 方法 | 给 HashMap 提供原始哈希 |
| **hash 值**   | 对 hashCode 扰动后的 int | HashMap 的 `hash()` 方法   | 用来算数组下标          |

**三者是层层递进的关系：**

```
key（你存的对象）→ hashCode（key.hashCode()）→ hash 值（扰动后）→ 数组下标
```

---

### 一个个拆开看

#### 1. key —— 你存进去的那个"键"

```java
Map<String, Integer> map = new HashMap<>();
map.put("张三", 90);
// key = "张三"（String 对象）
```

key 就是**你传进去的那个对象本身**，可以是 String、Integer、自定义对象等。

key 在 HashMap 里干两件事：
- **提供 hashCode**（调用 key 自身的 `hashCode()` 方法）
- **判断相等**（哈希冲突时，用 key 的 `equals()` 判断是不是同一个 key）

#### 2. hashCode —— key 自带的"身份证号"

```java
"张三".hashCode();  // 比如 774573
```

hashCode 是 **key 对象自己算出来的一个 int**。这是 Object 类就有的方法，每个类都可以重写自己的 hashCode 计算逻辑。

```java
// 不同对象的 hashCode 计算方式不同
String s = "abc";
s.hashCode();       // 96354（用 31 作为乘数算出来的）

Integer i = 42;
i.hashCode();       // 42（Integer 的 hashCode 就是它本身的值）

Object o = new Object();
o.hashCode();       // 一些随机数（默认是内存地址转成的 int）
```

**关键理解：hashCode 是属于 key 的，不是 HashMap 独有的。**

#### 3. hash 值 —— HashMap 对 hashCode 做了"二次加工"

```java
// HashMap 内部：拿到 hashCode 之后又加工了一步
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

**hash 值 = hashCode 的高 16 位和低 16 位做异或（XOR）的结果。**

为什么要多此一举？

```java
// 假设数组长度是 16，计算下标只用低 4 位
int index = hash & 15;  // 15 = 1111（二进制）
// 这意味着：hashCode 的高 28 位都废了，根本没参与下标计算
```

扰动就是为了让高位的信息也掺进低位：

```
原始 hashCode：     1101 0101 0110 0111 | 1001 0011 1010 0101
                  ↑ 高 16 位            ↑ 低 16 位

右移 16 位：        0000 0000 0000 0000 | 1101 0101 0110 0111

异或：              1101 0101 0110 0111 | 0100 0110 1100 0010
                                       ↑ 低 16 位混入了高位信息
```

**最终 hash 值的低 16 位既包含了自己原先的低位信息，也混入了高位的信息**，分布更均匀。

---

### 用一个具体的数字例子串起来

```java
String key = "张三";

// ① key 调用自己的 hashCode()
int h = key.hashCode();      // 假设计算出来是 774573

// ② HashMap 做扰动得到 hash 值
int hash = h ^ (h >>> 16);   
// 774573 右移 16 位后和原值异或

// ③ 用 hash 值计算数组下标（假设数组长度 16）
int index = hash & 15;       
// 得到 0~15 之间的一个数字
```

那 `hash` 和 `hashCode` 到底差多少？

```java
// hashCode 相同 → hash 也相同
String s1 = "Aa";            // hashCode = 2112
String s2 = "BB";            // hashCode = 2112（和 "Aa" 一样）
// 扰动后结果也一样

// hashCode 不同 → hash 不同 → 下标也可能相同（哈希冲突）
"abc".hashCode();            // 96354
"abcd".hashCode();           // 2987074
// 这两个取 & 15 之后可能相同（落在同一个桶）
```

---

### 面试常见追问

**问：两个不同 key 能有相同的 hashCode 吗？**

能。这就是 [[06Map接口.md#2 什么是 Hash 算法？|哈希冲突]]。比如 `"Aa"` 和 `"BB"` hashCode 都是 2112。

**问：两个不同 key 能有相同的 hash 值吗？**

能。hashCode 相同的话，扰动后的 hash 值也相同。

**问：两个不同 key 能有相同的数组下标吗？**

能！这就是**哈希冲突**。即使 hashCode 不同，取模后也可能落入同一个桶——因为数组长度有限，不同 hashCode 的低几位可能相同。

**问：那 hashCode 一样和下标一样，有什么区别？**

```
hashCode 相同 → 必然冲突，一定进同一个桶
hashCode 不同 → 也可能冲突（低位一样时取模结果相同）

所以：hashCode 不同不一定不冲突，但 hashCode 相同一定会冲突。
```

### 面试回答

> **面试官：hash 值、hashCode、key 有什么区别？**
>
> 这三者是层层递进的关系。key 是用户传进去的键对象，key 调用自己的 hashCode() 方法得到一个 int，这就是 hashCode。HashMap 拿到 hashCode 之后，会再做一次扰动（高 16 位和低 16 位异或），得到的结果才是 hash 值。最后用 hash 值跟数组长度做 `&` 运算，得到真正的数组下标。
>
> hashCode 和 hash 值的区别在于：hashCode 是 key 自己算的，属于"原始数据"；hash 值是 HashMap 加工过的，目的是让高位的特征也掺进低位，让哈希分布更均匀。如果数组长度比较小（比如默认 16），只用 hashCode 的低 4 位定下标，高位的信息就浪费了，扰动一下能让分布更均匀。

### 面试简答版

- **key**：用户传的键对象
- **hashCode**：key 自己算出来的 int（原始哈希）
- **hash 值**：HashMap 对 hashCode 扰动后的结果（`h ^ (h >>> 16)`）
- **关系**：key → hashCode → hash 值 → 数组下标
- **扰动目的**：让高位信息混入低位，分布更均匀

## 7. 什么是红黑树？

### 先讲场景

你已经知道 HashMap（JDK 8+）在链表长度 ≥ 8 时会转成红黑树。但红黑树到底是什么东西？为什么它能比链表快那么多？

```
链表查找：100 个元素挤在一个桶里，最差要看 100 次  → O(n)
红黑树查找：100 个元素最多看 7 次                    → O(log n)
```

红黑树就是一种"能自我平衡的二叉树"——它保证不管你怎么插入删除，查询效率始终是 O(log n)，不会退化。

---

### 先从简单的"二叉查找树"说起

二叉查找树（BST）的规则很简单：

```
左子树的所有节点 < 父节点 < 右子树的所有节点

      50 ← 根节点
     /  \
   30    70
  /  \     \
20    40    80

要找 40：50 比 40 大 → 走左边 → 30 比 40 小 → 走右边 → 找到了 ✅
每次比较砍掉一半，所以 BST 的查询是 O(log n)
```

但 BST 有个致命问题——**它会歪**：

```
按顺序插入：10 → 20 → 30 → 40 → 50

BST 变成一条直线：
10
  \
   20
     \
      30
        \
         40
            \
             50

这就退化成了链表，查询从 O(log n) 变成了 O(n)
```

所以我们需要一种**能自我纠正的 BST**——不管插入顺序是什么，它都不让树长歪。

---

### 红黑树怎么解决？——五条规则 + 管理员比喻

红黑树本质上还是二叉查找树，只是给每个节点加了**颜色**（红色或黑色），然后用五条规则来约束自己：

```
① 每个节点要么红要么黑
② 根节点是黑色
③ null 叶子节点视为黑色
④ 不能有连续的红节点（红节点的孩子必须是黑色）
⑤ 从根到任意叶子节点的路径上，黑色节点数量相同
```

听起来抽象，用个**管理员管队伍**的比喻来理解：

> 想象你有一支队伍（二叉查找树），高个子站右边，矮个子站左边。但有人乱站，队伍歪到一边去了。
>
> 你请了个管理员，定了几条规矩来保证队伍不走形：
>
> 1. **C 位（根节点）必须穿黑衣服**
> 2. **穿红衣服的人旁边不能挨着另一个穿红衣服的**（但可以挨黑衣服）
> 3. **从 C 位到任何一个空位，路上遇到的黑衣人数必须一样多**
>
> 每次有人插队进来，管理员就检查规则，不满足就让他**换个位置站**（旋转）或者**换件衣服**（变色）。

规则 ④（不连续红）和 ⑤（黑节点数相同）合起来的效果：

```
假设每条路径都有 3 个黑节点

最短路径（全是黑）：● → ● → ●（3 层）
最长路径（红黑交替）：● → ○ → ● → ○ → ●（5 层）

最长路径 ≤ 2 倍最短路径 → 树的高度最多差一倍
→ 查询不会退化到 O(n)，始终是 O(log n) ✅
```

**红黑树的平衡是一种"松散平衡"**——它不要求左右子树高度完全一样，只保证"最长路线不超过最短的 2 倍"。但这对性能来说已经足够了。

---

### 违规了怎么办？——修复手段

插入或删除节点时，五条规则可能被破坏。红黑树用两种手段来修复：

#### 1. 变色
```
红色 ↔ 黑色，互换颜色
```

#### 2. 旋转 —— 重新排列父子关系

旋转分两种：左旋和右旋，互为逆操作。

**左旋 — 右子节点升上来当父节点**

```
左旋前：           左旋后：
    父                 右子
   /  \               /   \
  A   右子      →     父    C
      /  \          / \
    左孙  C        A  左孙

变化：
  右子 → 新的父节点
  父   → 变成右子的左孩子
  左孙 → 变成父的右孩子
```

三步操作：
1. 把右子的左孩子（左孙）→ 过继给父，当父的右孩子
2. 把父 → 变成右子的左孩子
3. 把右子 → 升为新父节点

数值例子——对 50 做左旋：

```
左旋前：           左旋后：
    50                 70
   /  \               /   \
  30   70      →     50    80
      /  \          /  \
    65   80        30   65
```

**右旋 — 左子节点升上来当父节点**

```
右旋前：           右旋后：
    父                 左子
   /  \               /   \
 左子  C      →      A    父
 / \                       / \
A  右孙                  右孙  C

变化：
  左子 → 新的父节点
  父   → 变成左子的右孩子
  右孙 → 变成父的左孩子
```

三步操作：
1. 把左子的右孩子（右孙）→ 过继给父，当父的左孩子
2. 把父 → 变成左子的右孩子
3. 把左子 → 升为新父节点

数值例子——对 50 做右旋：

```
右旋前：           右旋后：
    50                 30
   /  \               /   \
  30   70      →     20    50
 / \                      / \
20  40                   40  70
```

**注意：旋转不会破坏 BST 的排序规则。** 左旋前后 `A < 父 < 左孙 < 右子 < C` 这个大小关系始终不变——旋转只是重新排列了父子关系，没有改变谁大谁小。

插入/删除的整体修复流程：

> **插进去 → 检查五条规则 → 违规了 → 变色/旋转 → 再检查 → 直到满足规则**

具体的修复情况分几种 case（叔叔节点是红还是黑、是不是 LL/RR/LR/RL 型），面试一般不问到这个深度，能说出"变色 + 旋转"就够了。

---

### 红黑树 vs AVL 树

AVL 树是另一种平衡树，比红黑树更"严格"——它要求左右子树高度差不能超过 1。

| 对比 | 红黑树 | AVL 树 |
|:---|:---|:---|
| 平衡标准 | 最长 ≤ 2 倍最短 | 左右子树高度差 ≤ 1 |
| 查询性能 | O(log n) | O(log n)，略快（更严格平衡） |
| 插入/删除 | **旋转少**，效率高 | 旋转多，效率低 |
| Java 使用 | HashMap、TreeMap、TreeSet | 没用 |

**Java 为什么选红黑树而不选 AVL 树？**

因为 HashMap 插入/删除非常频繁。红黑树的平衡条件更宽松，违规时需要旋转的次数比 AVL 树少。对于"插入删除多、查询也多"的场景，红黑树整体性能更好。

---

### 红黑树在 HashMap 中的具体应用

```java
// HashMap 的 TreeNode 就是红黑树节点
static final class TreeNode<K,V> extends LinkedHashMap.Entry<K,V> {
    TreeNode<K,V> parent;  // 父节点
    TreeNode<K,V> left;    // 左孩子
    TreeNode<K,V> right;   // 右孩子
    TreeNode<K,V> prev;    // 前一个节点（维护链表顺序）
    boolean red;           // 颜色：true=红, false=黑
}

// 链表长度 ≥ 8 且数组长度 ≥ 64 → 链表转红黑树
if (binCount >= TREEIFY_THRESHOLD - 1)  // TREEIFY_THRESHOLD = 8
    treeifyBin(tab, hash);

// 红黑树节点 ≤ 6 → 退化回链表
if (lc <= UNTREEIFY_THRESHOLD)  // UNTREEIFY_THRESHOLD = 6
    untreeify(loHead);
```

**为什么转树是 8，退化是 6？**

中间留了 2 个数的缓冲，防止频繁插入删除导致反复在链表和树之间切换（性能抖动）。源码里叫"防抖"设计。

---

### 面试回答

> **面试官：什么是红黑树？**
>
> 红黑树是一种自平衡的二叉查找树，每个节点有红色或黑色。它通过五条规则来保持平衡：根节点是黑色的、不能有连续的红节点、每条路径上黑色节点数量相同。这些规则保证了最长路径不超过最短路径的 2 倍，所以查找、插入、删除的时间复杂度都是 O(log n)。
>
> 当插入或删除破坏了规则时，红黑树通过变色和旋转来修复。相比 AVL 树，红黑树的平衡条件更宽松，旋转次数更少，更适合插入删除频繁的场景。
>
> HashMap 在链表长度 ≥ 8 时把链表转成红黑树，防止哈希冲突严重时查询退化成 O(n)。TreeMap 底层也是直接用红黑树实现的。

### 面试简答版

- **红黑树** = 能自我平衡的二叉查找树，节点分红/黑
- **五条规则**：根黑、无连续红节点、各路径黑节点数相同
- **平衡效果**：最长路径 ≤ 2 倍最短路径 → O(log n)
- **修复手段**：变色 + 旋转（左旋/右旋）
- **vs AVL 树**：红黑树旋转更少，Java 选用红黑树
- **HashMap 用途**：链表 ≥ 8 转红黑树兜底，防止 O(n) 退化

## 8. HashMap 的 put 方法具体流程？

### 先讲场景

第 4 题给了 put 的大概流程，但面试问这道题时，是要你把**每一步判断、每个分支、每个细节**都说明白。

```java
map.put("张三", 90);
```

这行代码背后，JDK 8 的 putVal 方法走了整整**五个大步骤**，每一步都有分支判断。下面直接对着源码逐行拆。![[Pasted image 20260502203919.png]]

---

### 先看完整代码（JDK 8）

```java
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
               boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    
    // ① 懒加载：table 为空则初始化
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    
    // ② 定位下标，空位直接放
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    else {
        Node<K,V> e; K k;
        
        // ③ 头节点就是目标 key → 准备覆盖
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
        
        // ④ 树节点 → 走红黑树的插入逻辑
        else if (p instanceof TreeNode)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        
        // ⑤ 链表 → 遍历查找或尾插
        else {
            for (int binCount = 0; ; ++binCount) {
                if ((e = p.next) == null) {      // 遍历到尾部都没找到 → 尾插
                    p.next = newNode(hash, key, value, null);
                    if (binCount >= TREEIFY_THRESHOLD - 1) // 检查是否转树
                        treeifyBin(tab, hash);
                    break;
                }
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;  // 在链表里找到了 → 准备覆盖
                p = e;
            }
        }
        
        // ⑥ 覆盖旧值
        if (e != null) {
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            return oldValue;
        }
    }
    
    // ⑦ 修改计数器 + 检查扩容
    ++modCount;
    if (++size > threshold)
        resize();
    afterNodeInsertion(evict);
    return null;
}
```

---

### 分步拆解

#### ① 懒加载——第一次 put 才初始化数组

```java
if ((tab = table) == null || (n = tab.length) == 0)
    n = (tab = resize()).length;
```

HashMap 的数组（table）不是 new 出来就创建的，而是**第一次 put 时通过 resize() 初始化**。这叫**懒加载**，省内存。

```java
Map<String, Integer> map = new HashMap<>();
// 此时 table = null，没分配任何数组

map.put("张三", 90);
// 第一次 put → table 为空 → 调 resize() → 创建长度 16 的数组
```

#### ② 定位下标——空位直接放

```java
if ((p = tab[i = (n - 1) & hash]) == null)
    tab[i] = newNode(hash, key, value, null);
```

用 `(n - 1) & hash` 算出数组下标，如果那个位置**还空着**，直接 new 一个 Node 放进去。这是最理想的情况——**O(1)**，没有冲突。

```
        数组                          数组
        ┌────┐                        ┌────┐
        │    │                        │    │
        ├────┤                        ├────┤
下标 i  │    │      put("张三",90)    │ 张 │  ← 直接放进去 ✅
        ├────┤       ──────────→      ├────┤
        │    │                        │    │
        └────┘                        └────┘
```

#### ③ 头节点 equals = true → 直接覆盖

```java
if (p.hash == hash &&
    ((k = p.key) == key || (key != null && key.equals(k))))
    e = p;
```

如果下标位置**已经有元素了**，先检查这个"头节点"是不是同一个 key。判断逻辑：

```
1. 先比 hash（快速筛选，hash 不同 key 一定不同）
2. 再比 ==（引用相同就不用调 equals 了）
3. 最后调 equals（真正的业务相等判断）
```

**先比 hash 再比 equals 是一个性能优化：** hash 比较只是一个 int 的 ==，比调 equals 快得多。hash 对不上直接跳过，不用走 equals。

#### ④ 红黑树节点 → 走树的插入

```java
else if (p instanceof TreeNode)
    e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
```

如果头节点是 `TreeNode` 类型，说明这个桶已经是红黑树了，走 `putTreeVal()` 在树里插入或覆盖。

树的插入过程：按照 BST 的规则（hash 比大小）找到位置插入，然后通过变色 + 旋转修复红黑树的五条规则。

#### ⑤ 链表 → 遍历 + 尾插

```java
else {
    for (int binCount = 0; ; ++binCount) {
        if ((e = p.next) == null) {          // 走到链表尾部
            p.next = newNode(...);           // 尾插
            if (binCount >= TREEIFY_THRESHOLD - 1) // ≥ 8 时转树
                treeifyBin(tab, hash);
            break;
        }
        if (e.hash == hash && equals)         // 在链表中找到了
            break;
        p = e;
    }
}
```

链表插入分两种情况：

**情况 A：遍历到链表末尾都没找到 → 直接尾插**

```
原本链表：A → B → C → null
插入 D：   A → B → C → D → null  ✅
```

**情况 B：在链表中找到了相同 key → 准备覆盖**

```
原本链表：A → B → C → null（要找 "张三"）
发现 B 就是 "张三" → 走到步骤 ⑥ 覆盖 value
```

**插完后检查链表长度：如果 ≥ 8，调 treeifyBin 尝试转树。**

注意 `treeifyBin` 里还有一个判断：

```java
// treeifyBin 源码开头
if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY)
    resize();  // 数组还太小 → 优先扩容，不转树（扩容机制见 [[06Map接口.md#扩容机制|4. 扩容机制]]）
// 数组 ≥ 64 → 才真的转树
```

所以完整条件是：**链表 ≥ 8 且数组 ≥ 64 → 转树。**

#### ⑥ 覆盖旧值

```java
if (e != null) {
    V oldValue = e.value;
    if (!onlyIfAbsent || oldValue == null)
        e.value = value;
    afterNodeAccess(e);
    return oldValue;
}
```

如果前面找到了相同 key（e != null），这里覆盖 value。`onlyIfAbsent` 参数对应 `putIfAbsent()` 方法——为 true 时只有原 value 为 null 才覆盖。

覆盖完成后**返回旧值**（不是新值）。

#### ⑦ 最后检查扩容

```java
++modCount;
if (++size > threshold)
    resize();
```

这是 put 的最后一步：

- **modCount** +1：记录结构修改次数，给 fail-fast 用的（[[03集合容器概述.md#9 Java集合的快速失败机制 fail-fast|回顾 fail-fast]]）
- **size** +1：元素个数
- 如果 `size > threshold`（容量 × 负载因子）→ **扩容**（详细过程见 [[06Map接口.md#扩容机制|4. 扩容机制]]，JDK 1.7/1.8 区别见 [[06Map接口.md#区别三扩容时的 rehash 方式|5. 扩容 rehash 区别]]）

注意：**覆盖 value 不算结构修改**，只有真正新增元素才 +1 扩容。这就是为什么 modCount 和覆盖的 return 之间隔开了。

---

### 完整的 put 流程图

```
put(key, value)
  │
  ├─ ① table == null？ → resize() 初始化数组
  │
  └─ ② i = (n - 1) & hash
       │
       ├─ tab[i] == null → new Node 放进去 ✅
       │
       └─ tab[i] != null → 冲突
            │
            ├─ hash 相同 && equals → e = 头节点
            │
            ├─ p instanceof TreeNode → putTreeVal() 树插入
            │
            └─ 遍历链表
                 ├─ 找到相同 key → e = 该节点
                 └─ 到尾部没找到 → 尾插 newNode
                      └─ 链表 ≥ 8 → treeifyBin()
                           ├─ 数组 < 64 → resize() 扩容
                           └─ 数组 ≥ 64 → 转红黑树
       │
       └─ ⑥ e != null → 覆盖 value，返回旧值 ✅
       │
       └─ ⑦ ++modCount, ++size
            └─ size > threshold → resize()
```

---

### 几个容易答漏的细节

**1. put 返回什么？**

```java
// 覆盖旧 key → 返回旧 value
map.put("A", 1);
System.out.println(map.put("A", 2));  // 输出 1（返回旧值）

// 新增 key → 返回 null
System.out.println(map.put("B", 1));  // 输出 null
```

**2. modCount 什么时候加？**

只有**真正新增了元素**才加（结构修改）。覆盖 value 不算结构修改，modCount 不变。

**3. TREEIFY_THRESHOLD = 8 是怎么算的？**

源码上说：基于泊松分布，在随机 hash 下链表长度到 8 的概率只有 **亿分之六**。所以 ≥ 8 意味着 hash 出了严重问题，需要红黑树兜底。

### 面试回答

> **面试官：说一下 HashMap 的 put 流程？**
>
> put 方法分七个步骤。
>
> 第一，检查 table 是否为空，空则调 resize() 初始化（懒加载）。
>
> 第二，用 `(n - 1) & hash` 定位数组下标，如果该位置为 null，直接 new Node 放进去。
>
> 第三，如果位置不为 null，先比较头节点的 hash 和 equals，相同就直接准备覆盖。
>
> 第四，如果头节点是 TreeNode，走红黑树的 putTreeVal 插入。
>
> 第五，如果头节点是普通链表节点，遍历链表。找到相同 key 就准备覆盖，遍历到末尾都没找到就尾插。插完后检查链表长度 ≥ 8，调 treeifyBin——但如果数组还小于 64，会优先扩容而不是转树。
>
> 第六，如果在前面找到了相同 key，覆盖 value 并返回旧值。
>
> 第七，如果确实是新增元素，modCount 加一，size 加一，检查是否超过 threshold，超过就 resize 扩容。

### 面试简答版

- **七步**：① 懒加载 → ② 空位直接放 → ③ 头节点冲突 → ④ 树插入 → ⑤ 链表遍历/尾插 → ⑥ 覆盖旧值 → ⑦ 检查扩容
- **冲突处理**：先比 hash 快速筛选，再比 equals 确认
- **链表→树**：链表 ≥ 8 且数组 ≥ 64 才转树，不够 64 就优先扩容
- **覆盖 vs 新增**：覆盖返回旧值、不改 modCount、不触发扩容检查（扩容细节见 [[06Map接口.md#扩容机制|4. 扩容机制]]）；新增才++size、检查 threshold
- **树插入**：走 TreeNode.putTreeVal，按 BST 定位后变色+旋转修复红黑树规则

## 9. HashMap 的扩容操作是怎么实现的？

### 先讲场景

你在第 4 题和第 8 题都看到了"扩容"两个字：put 时如果 `size > threshold`，就调 `resize()`。但 resize 到底干了什么？它怎么把旧数组的数据搬到新数组？JDK 8 为什么说扩容性能比 JDK 7 好？

这道题就专门拆开讲。

**一句话概括：新建一个 2 倍大的数组，把旧数组每个桶里的元素重新散列到新数组里。**

---

### 什么时候触发扩容？

```java
// 扩容触发条件
if (++size > threshold)     // size 是元素个数，threshold 是阈值
    resize();

// threshold = capacity * loadFactor
// 默认：16 * 0.75 = 12
// 也就是说元素超过 12 个时，数组翻倍到 32，新 threshold = 32 * 0.75 = 24
```

注意：**只有真正新增元素时才会检查扩容**，覆盖旧 value 不触发。

---

### JDK 8 resize() 完整流程

```java
final Node<K,V>[] resize() {
    Node<K,V>[] oldTab = table;
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    int oldThr = threshold;
    int newCap, newThr = 0;
    
    // ① 计算新容量和新阈值
    if (oldCap > 0) {
        if (oldCap >= MAXIMUM_CAPACITY) {          // 到上限了
            threshold = Integer.MAX_VALUE;          // 不再扩容
            return oldTab;
        }
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY)  // 翻倍
            newThr = oldThr << 1;                              // 阈值也翻倍
    }
    else if (oldThr > 0)   // 初始容量设为了 threshold（指定了初始大小时）
        newCap = oldThr;
    else {                  // 第一次 put，用默认值
        newCap = DEFAULT_INITIAL_CAPACITY;  // 16
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);  // 12
    }
    
    // ② 创建新数组
    Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    table = newTab;
    
    // ③ 搬数据（核心）
    if (oldTab != null) {
        for (int j = 0; j < oldCap; j++) {
            Node<K,V> e = oldTab[j];
            if (e != null) {
                oldTab[j] = null;  // 帮 GC
                
                if (e.next == null)           // 情况 1：只有单个节点
                    newTab[e.hash & (newCap - 1)] = e;
                    
                else if (e instanceof TreeNode)  // 情况 2：红黑树
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                    
                else {                          // 情况 3：链表
                    // 分成低位链表和高位链表
                    Node<K,V> loHead = null, loTail = null;
                    Node<K,V> hiHead = null, hiTail = null;
                    Node<K,V> next;
                    do {
                        next = e.next;
                        if ((e.hash & oldCap) == 0) {  // 留在原位
                            if (loTail == null)
                                loHead = e;
                            else
                                loTail.next = e;
                            loTail = e;
                        }
                        else {                          // 挪到 原位+oldCap
                            if (hiTail == null)
                                hiHead = e;
                            else
                                hiTail.next = e;
                            hiTail = e;
                        }
                        e = next;
                    } while (e != null);
                    
                    if (loTail != null) {
                        loTail.next = null;
                        newTab[j] = loHead;           // 低位链表放原位
                    }
                    if (hiTail != null) {
                        hiTail.next = null;
                        newTab[j + oldCap] = hiHead;  // 高位链表放 原位+oldCap
                    }
                }
            }
        }
    }
    return newTab;
}
```

---

### 逐个拆解

#### ① 计算新容量

有三种情况：

```java
// 情况 1：已经初始化过 → 正常扩容（翻倍）
if (oldCap > 0) {
    newCap = oldCap << 1;        // 左移一位 = 乘 2
    newThr = oldThr << 1;        // 阈值也翻倍
}

// 情况 2：指定了初始容量 → 用 threshold 当容量
// 比如 new HashMap(32) → threshold 被设为 32
else if (oldThr > 0)
    newCap = oldThr;

// 情况 3：第一次 put，啥都没设 → 默认 16 和 12
else {
    newCap = 16;
    newThr = 12;
}
```

#### ② 搬数据的三种情况

搬数据时，旧数组的每个桶有三种可能：

**情况 A：桶里只有一个节点**
```java
if (e.next == null)
    newTab[e.hash & (newCap - 1)] = e;
```
直接算新下标，放过去就行。最简单。

**情况 B：桶里是红黑树**
```java
else if (e instanceof TreeNode)
    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
```
走 TreeNode.split()，同样按 `(e.hash & oldCap)` 分成低位和高位两条链，如果拆完后树太小（≤ 6）就退化成链表。

**情况 C：桶里是链表 —— 核心优化**
```java
else {
    Node<K,V> loHead = null, loTail = null;  // 低位链表（留在原位）
    Node<K,V> hiHead = null, hiTail = null;  // 高位链表（挪到原位+oldCap）
    do {
        next = e.next;
        if ((e.hash & oldCap) == 0) {
            // → 加入低位链表（原位）
            ...
        } else {
            // → 加入高位链表（原位 + oldCap）
            ...
        }
        e = next;
    } while (e != null);
    
    newTab[j] = loHead;            // 低位链表放回原位
    newTab[j + oldCap] = hiHead;   // 高位链表放到新位置
}
```

这是 JDK 8 扩容最精妙的设计，下面单独讲。

---

### 核心优化：`e.hash & oldCap` 判断不用重新算 hash

JDK 7 扩容时，每个元素都要重新算一次下标：

```java
// JDK 7：重新算
int newIndex = e.hash & (newCapacity - 1);
```

JDK 8 换了个思路——容量翻倍后，一个元素的新下标只有两种可能：**原位** 或 **原位 + 旧容量**。

为什么？

```
旧容量 16（二进制 10000），新容量 32（二进制 100000）

数组下标计算公式：hash & (length - 1)

旧下标 = hash & 15  （只取低 4 位）
新下标 = hash & 31  （取低 5 位）

其实就是多看了 hash 的第 5 位（从右数）：
  - 第 5 位 = 0 → hash & 31 和 hash & 15 的结果一样 → 留原位
  - 第 5 位 = 1 → hash & 31 = hash & 15 + 16 → 挪到 原位 + 16
```

用代码判断就是：

```java
// oldCap = 16（二进制 10000）
if ((e.hash & oldCap) == 0) {
    // 第 5 位是 0 → 留原位
} else {
    // 第 5 位是 1 → 挪到 原位 + oldCap
}
```

用一个具体例子：

```java
// 假设旧容量 16

元素 A：hash = 5     → 二进制 00101
  旧下标 = 5 & 15  = 5
  hash & 16 = 5 & 16 = 0  → 第 5 位 = 0 → 新下标 = 5（原位 ✅）

元素 B：hash = 21    → 二进制 10101
  旧下标 = 21 & 15 = 5（和 A 冲突了）
  hash & 16 = 21 & 16 = 16 ≠ 0  → 第 5 位 = 1 → 新下标 = 5 + 16 = 21
```

你看，原来冲突在同一个桶的 A 和 B，扩容后自动分到了两个不同的位置——**扩容顺便解决了部分哈希冲突**。

---

### 为什么 JDK 8 扩容性能更好？

| 对比 | JDK 7 | JDK 8 |
|:---|:---|:---|
| 计算方式 | 每个元素重新 `hash & (newCap - 1)` | `hash & oldCap` 位运算，更少运算 |
| 链表顺序 | 头插法，**顺序反转** | 尾插法，**顺序不变** |
| 并发安全 | 可能死循环 | 避免死循环（但仍非线程安全） |
| 红黑树 | 无 | 树节点拆分成两条链，自动退化 |

---

### 面试回答

> **面试官：HashMap 的扩容是怎么实现的？**
>
> 扩容发生在 put 时检查到 size > threshold（容量 × 负载因子），每次扩容为原来的 2 倍。
>
> 扩容分三步：创建新数组、遍历旧数组每个桶搬数据、把桶里的元素重新散列到新数组。
>
> 搬数据时分三种情况：单节点直接算新下标放过去；链表用 `e.hash & oldCap` 分成低位和高位两条链，低位留在原位，高位挪到"原位 + 旧容量"的位置；红黑树也是同样的逻辑拆分，拆完如果树太小就退化成链表。
>
> JDK 8 的优化在于：`e.hash & oldCap` 只做一次位运算就能确定新位置，比 JDK 7 重新计算每个元素的下标效率更高，而且不会反转链表顺序，避免了并发死循环的问题。另外，扩容时原本冲突的元素可能会被分到不同位置，相当于顺便缓解了哈希冲突。

### 面试简答版

- **触发条件**：`size > threshold`（默认 16 × 0.75 = 12）
- **扩容方式**：每次翻倍（`oldCap << 1`）
- **搬数据核心**：`e.hash & oldCap` 判断原位还是挪位
  - `= 0` → 留原位
  - `≠ 0` → 挪到 `原位 + oldCap`
- **三种情况**：单节点直接放、链表拆高低两条链、红黑树 split 拆分
- **JDK 8 优势**：不用重新算 hash、顺序不变、无死循环、拆分红黑树后自动退化

## 10. HashMap 是怎么解决哈希冲突的？

### 先讲场景

哈希冲突（也叫哈希碰撞）就是：**两个不同的 key，算出了相同的数组下标**。

```java
// 经典冲突例子
"Aa".hashCode();   // 2112
"BB".hashCode();   // 2112 ← 一样！

// 或者 hashCode 不同但低位相同
hashA = 5;         // 二进制 00101
hashB = 21;        // 二进制 10101
// 数组长度 16 时：5 & 15 = 5，21 & 15 = 5 → 都落在下标 5
```

**哈希冲突无法完全避免**（抽屉原理：16 个抽屉装 100 个元素，必然有抽屉装多个）。HashMap 解决冲突分两个层面：

```
① 直接解决 → 冲突了怎么存？（拉链法：链表/红黑树）
② 预防减少 → 尽量让冲突少发生（扰动函数、2 的幂容量、负载因子）
```

---

### 直接解决方式：拉链法

拉链法的核心思路：**数组的每个位置不直接存元素，而是存一个链表（或红黑树）的头节点。冲突了？链起来就行。**

```
数组 + 链表/红黑树：

下标 5：［ ］→［Aa］→［BB］→［Cc］→ null
        └─────── 拉链 ────────┘
        同一个桶里的多个 key，串成一条链
```

不同 JDK 版本实现不同：

| 版本 | 冲突时的存储 | 插入方式 | 什么时候转树 |
|:---|:---|:---|:---|
| JDK 7 | 链表 | **头插法**（新节点插链表头） | 不转树，永远链表 |
| JDK 8 | 链表 → **红黑树** | **尾插法**（新节点挂链表尾） | 链表 ≥ 8 且数组 ≥ 64 |

具体细节可以回顾 [[06Map接口.md#5 HashMap 在 JDK 1.7 和 1.8 中有哪些不同？还有哪些集合有类似更新？|第 5 题 JDK 7 vs 8 区别]] 和 [[06Map接口.md#7 什么是红黑树？|第 7 题红黑树]]。

---

### 冲突了怎么找到对的 key？

冲突之后，同一个桶里有多个 key。get 的时候怎么知道自己要找的是哪个？

答案是**两步判断**：

```java
// put 或 get 时，在链表中找 key 的逻辑
if (e.hash == hash &&                    // 第一步：比 hash（快速筛选）
    ((k = e.key) == key ||               // 第二步：比引用
     (key != null && key.equals(k))))    // 或比 equals
```

```
先用 hash 快速过滤（hash 不同一定不是同一个 key）
再用 equals 精确确认（hash 相同也不一定是同一个 key）
```

**为什么先比 hash 再比 equals？** 因为 hash 只是一个 int 的 `==` 比较，比调 equals 快得多。能提前筛掉大部分不相关的 key，减少 equals 的调用次数。

这部分在 [[06Map接口.md#8 HashMap 的 put 方法具体流程？|第 8 题 put 流程]] 的步骤 ③ 和 ⑤ 有详细代码。

---

### 预防措施一：扰动函数（让 hash 分布更均匀）

既然冲突是从 hashCode 到数组下标的映射不可控导致的，那能不能让这个映射更均匀？

```java
// JDK 8 的扰动函数
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

让高 16 位和低 16 位做异或，把高位的信息掺进低位。这样仅靠低位决定数组下标时，也能用上高位的信息，分布更均匀。

具体原理见 [[06Map接口.md#6 hash 值、hashCode、key 有什么区别？|第 6 题 hash 值 vs hashCode]]。

---

### 预防措施二：容量为 2 的幂（让下标计算更均匀）

```java
// HashMap 计算下标
int index = (n - 1) & hash;  // n 是数组长度
```

如果 `n` 是 2 的幂，`n - 1` 的二进制全是 1，`&` 运算的结果等同于 `hash % n`，且每个位置都有同等概率被选中。

```
n = 16:  n - 1 = 1111（二进制）
hash & 1111 = 保留低 4 位，范围 0~15

n = 15:  n - 1 = 1110（二进制）
hash & 1110 = 最低位永远为 0
→ 奇数位置永远不会被选中 → 空间浪费一半，冲突概率翻倍
```

所以 HashMap 强制容量为 2 的幂。如果你传 `new HashMap(15)`，它会自动给你调整成 16。

细节见 [[06Map接口.md#4 说一下 HashMap 的实现原理？|第 4 题 为什么要用 & 而不是 %]]。

---

### 预防措施三：负载因子 0.75（空间和时间的折中）

```java
// 默认 16 * 0.75 = 12
// 元素超过 12 个就扩容，数组变大后元素重新散列，冲突减少
```

- 负载因子越**小**（如 0.5）→ 冲突少、查询快，但空间浪费多、频繁扩容
- 负载因子越**大**（如 1）→ 空间利用率高，但冲突多、链表长、查询慢
- **0.75** 是经过大量测试的平衡点

这也是 [[06Map接口.md#4 说一下 HashMap 的实现原理？|第 4 题]] 里提到过的。

---

### 扩容带来的"被动缓解"

即使做了预防，冲突还是会发生。但扩容时会顺带缓解冲突：

```java
// 旧容量 16 时，hash=5 和 hash=21 都落在下标 5
// 扩容到 32 后：
元素 A（hash=5）： 5 & 31 = 5   → 留原位
元素 B（hash=21）：21 & 31 = 21 → 挪到下标 21 ✅ 分开了
```

扩容相当于给元素**重新分座位**——原本挤在一起的可能就分开了。这就是 [[06Map接口.md#9 HashMap 的扩容操作是怎么实现的？|第 9 题扩容]] 里讲的 `e.hash & oldCap` 的效果。

---

### 一张图总结

```
哈希冲突解决策略
│
├─ 直接解决：拉链法
│   ├─ JDK 7：数组 + 链表（头插法）
│   └─ JDK 8：数组 + 链表/红黑树（尾插法）
│   └─ 找 key：先比 hash 快速过滤，再比 equals 精确确认
│
└─ 预防减少
    ├─ 扰动函数（高 16 位 ⊕ 低 16 位）
    ├─ 容量为 2 的幂（下标计算均匀）
    ├─ 负载因子 0.75（适时扩容）
    └─ 扩容时元素重新散列（被动缓解）
```

### 面试回答

> **面试官：HashMap 是怎么解决哈希冲突的？**
>
> 分两个层面。直接解决方式是拉链法——数组每个位置存一个链表（或红黑树）的头节点，冲突的元素链在一起。JDK 8 中，链表超过 8 个且数组达到 64 时会转成红黑树，保证查询效率。在链表中查找时，先比 hash 快速过滤，再用 equals 精确确认。
>
> 预防层面还有三个设计来减少冲突：一是扰动函数，把 hashCode 的高 16 位和低 16 位异或，让高位也参与下标计算；二是容量强制为 2 的幂，让 `(n - 1) & hash` 的分布均匀；三是负载因子 0.75，在空间和性能之间取平衡。
>
> 另外，扩容时元素会重新散列，原本冲突的 key 可能被分到不同位置，相当于顺带缓解了冲突。

### 面试简答版

- **直接解决**：拉链法（数组 + 链表/红黑树）
- **找 key**：先比 hash 快速过滤 → 再比 equals 精确确认
- **预防 ① 扰动函数**：`h ^ (h >>> 16)` 让高位掺进低位
- **预防 ② 2 的幂容量**：`(n-1) & hash` 均匀分布
- **预防 ③ 负载因子 0.75**：适时扩容减少冲突
- **被动缓解**：扩容时元素重散列，可能把冲突的 key 分开

## 11. 能否使用任何类作为 Map 的 key？

### 先讲场景

看这段代码，你觉得有问题吗？

```java
class Student {
    String id;
    String name;
    // 没写 hashCode，也没写 equals
}

Map<Student, Integer> map = new HashMap<>();
map.put(new Student("001", "张三"), 90);
System.out.println(map.get(new Student("001", "张三")));  // null 🤯
```

取出来是 null。为什么？因为 `new Student("001", "张三")` 每次 new 都是不同的对象，默认的 hashCode 也不一样，所以两次 put/get 定位到了不同的桶。

所以回答这道题：**谁都能当 key，但能不能"当好"是另一回事。**

---

### 核心要求：必须重写 hashCode + equals

这是最基本的。HashMap 靠 hashCode 定位桶，靠 equals 在桶里找对的 key。

```java
// HashMap 的 get 流程
int hash = hash(key);           // 调用 key.hashCode()
int index = (n - 1) & hash;     // 定位桶
// 在桶里遍历，比较 key.equals(k)
```

**不重写 hashCode 会怎样？**

```java
class Key {
    String value;
    Key(String value) { this.value = value; }
    // ❌ 没重写 hashCode → 继承 Object 的默认实现
    // 每个 new Key("abc") 的 hashCode 都不一样
}

Map<Key, String> map = new HashMap<>();
map.put(new Key("a"), "1");
map.get(new Key("a"));  // null → hashCode 不同，定位到不同桶
```

**只重写 equals 不重写 hashCode 会怎样？**

```java
class Key {
    String value;
    Key(String value) { this.value = value; }
    
    @Override
    public boolean equals(Object o) {
        return o instanceof Key k && Objects.equals(this.value, k.value);
    }
    // ❌ 没重写 hashCode → equals 为 true 的两个对象 hashCode 不同
    // 违反了 hashCode 约定：equals 相等 → hashCode 必须相等
}

Map<Key, String> map = new HashMap<>();
map.put(new Key("a"), "1");
map.get(new Key("a"));
// hashCode 不同 → 定位到不同桶 → 找不到，返回 null
// 哪怕两个 key 的 equals 返回 true
```

**正确做法：**

```java
class Key {
    String value;
    Key(String value) { this.value = value; }
    
    @Override
    public boolean equals(Object o) {
        return o instanceof Key k && Objects.equals(this.value, k.value);
    }
    
    @Override
    public int hashCode() {
        return Objects.hash(value);  // 用和 equals 相同的字段算
    }
    // ✅ 现在 equals 相等的两个 key，hashCode 也相等
    // → 定位到同一个桶 → get 能找到
}
```

这部分可以回顾 [[05Set接口.md#3 hashCode 和 equals 的规定|05 Set 接口的 hashCode/equals 规定]]。

---

### 最佳实践：用不可变类做 key

就算重写了 hashCode 和 equals，还有一个坑：**不要修改放入 Map 后的 key**。

```java
Map<List<String>, String> map = new HashMap<>();

List<String> key = new ArrayList<>();
key.add("a");
map.put(key, "value1");  // 此时 key 的 hashCode 是 ["a"] 算出来的

key.add("b");            // ❌ 修改了 key
// hashCode 变了！但 HashMap 不知道，它还在用旧的 hashCode 记录的位置

map.get(key);            // → null（定位到新位置，但旧位置才是存放的地方）
map.get(originalKey);    // → 也找不到
```

这就是为什么 **String、Integer 这些不可变类是最佳的 key**——它们的 hashCode 永远不会变，不存在"放进去之后 key 变了"的问题。

```java
// ✅ String 是不可变的
Map<String, Integer> map = new HashMap<>();
map.put("hello", 1);  // 永远不用担心 "hello" 这个字符串会变

// ⚠️ 自定义可变对象需要小心
Map<Student, Integer> map = new HashMap<>();
map.put(student, 90);
student.setId("002");  // ❌ 别再改 student 了！找不到了
```

---

### TreeMap 的特殊要求

TreeMap 不靠 hashCode 定位，而是靠**排序**，所以要求不太一样：

```java
// TreeMap 要求 key 要么实现了 Comparable 接口
Map<String, Integer> treeMap = new TreeMap<>();  // ✅ String 实现了 Comparable

// 要么在构造时传入 Comparator
Map<Student, Integer> map = new TreeMap<>((a, b) -> a.id.compareTo(b.id));  // ✅

// 什么都没提供 → 抛异常 ClassCastException ❌
```

HashMap 靠 hashCode/equals 找 key，TreeMap 靠 Comparable/Comparator 比大小。两种不同的"找 key"机制。

---

### 总结

| 要求 | HashMap / LinkedHashMap | TreeMap |
|:---|:---|:---|
| 必须的 | **hashCode + equals** | **Comparable** 或传 **Comparator** |
| 推荐 | 不可变类 | 不可变类 |
| 典型好 key | String、Integer、枚举 | String、Integer |

### 面试回答

> **面试官：能否使用任何类作为 Map 的 key？**
>
> 从语法上讲任何对象都可以做 key，但要让 Map 正常工作，必须满足对应 Map 的要求。
>
> 对于 HashMap，key 必须正确重写 hashCode 和 equals 方法——hashCode 负责定位桶，equals 负责在桶里找到对的 key。如果只重写了一个，或者两个都没重写，put 进去之后就取不出来了。
>
> 另外，最好用不可变类做 key，比如 String、Integer。如果用可变对象做 key，放进去之后就不要修改它了，否则 hashCode 会变，导致再也找不到。
>
> 对于 TreeMap 要求不一样，它不依赖 hashCode，而是要求 key 实现 Comparable 接口，或者在构造时传入 Comparator。

### 面试简答版

- **任何类都可以做 key，但要正确工作必须满足要求**
- **HashMap 要求**：重写 `hashCode` + `equals`
- **常见错误**：没重写 hashCode / 只重写 equals / 修改 key 后 hashCode 变了
- **最佳 key**：**不可变类**（String、Integer、枚举）
- **TreeMap 要求**：实现 `Comparable` 或传入 `Comparator`
- **⚠️ 绝对不要**：put 进去之后修改 key 对象

## 12. 为什么 HashMap 中 String、Integer 这样的包装类适合作为 key？

### 先讲场景

上一题说了"推荐用 String、Integer 等不可变类做 key"，但具体好在哪里？用 `String` 做 key 和用自己写的 `Student` 类做 key，到底差在哪？

说白了就是四个字：**省心、安全、快**。

下面从三个角度拆开看。

---

### 原因一：不可变性——hashCode 永不改变

这是最核心的原因。String 和 Integer 一旦创建，内部的值**永远不变**。

```java
String key = "hello";
key.hashCode();  // 99162322
// 不管过多久，调多少次，"hello".hashCode() 永远是 99162322

Integer key = 42;
key.hashCode();  // 42
// Integer 的 hashCode 就是它自身，永远不会变
```

这意味着：**放进去的时候 hashCode 是多少，取的时候还是多少。** 定位到同一个桶，一定能找到。

对比一下可变对象：

```java
// 可变 key 的惨案
StringBuilder key = new StringBuilder("a");
Map<StringBuilder, String> map = new HashMap<>();
map.put(key, "value1");     // 此时 key 的 hashCode 是 "a" 算出来的

key.append("b");             // ❌ 修改了 key
// hashCode 变了！变成 "ab" 算出来的值

map.get(key);                // null → 定位到了新桶，但数据在旧桶
map.get(new StringBuilder("a")); // 还是 null → 旧桶的 hash 也对不上
```

**String 不会出现这个问题。** 你放一个 `"hello"` 进去，它永远都是 `"hello"`，没人能改它。

---

### 原因二：hashCode 和 equals 已正确重写——拿来就用

如果你自己写一个类做 key，你得自己操心两件事：

```java
// 自定义 key：你得自己保证写对了
class Student {
    String id;
    
    @Override
    public boolean equals(Object o) { ... }  // 自己写，容易出错
    
    @Override
    public int hashCode() { ... }            // 自己写，要和 equals 保持一致
}
```

用 String、Integer 则完全不用操这个心——**JDK 已经帮你写好了，保证正确：**

```java
// String 的 hashCode——用 31 做乘数，分布均匀
public int hashCode() {
    int h = hash;
    if (h == 0 && value.length > 0) {
        for (char v : value) {
            h = 31 * h + v;
        }
        hash = h;
    }
    return h;
}

// String 的 equals——逐字符比较
public boolean equals(Object anObject) {
    if (this == anObject) return true;
    if (anObject instanceof String) {
        String anotherString = (String)anObject;
        int n = value.length;
        if (n == anotherString.value.length) {
            char v1[] = value;
            char v2[] = anotherString.value;
            int i = 0;
            while (n-- != 0) {
                if (v1[i] != v2[i]) return false;
                i++;
            }
            return true;
        }
    }
    return false;
}

// Integer 的 hashCode——直接返回自身值，简单高效
@Override
public int hashCode() {
    return Integer.hashCode(value);
}
public static int hashCode(int value) {
    return value;  // 就是自己，还能更简单吗
}

// Integer 的 equals——比较数值
public boolean equals(Object obj) {
    if (obj instanceof Integer) {
        return value == ((Integer)obj).intValue();
    }
    return false;
}
```

**两个都正确重写了 hashCode 和 equals，且保持了一致：**
- `"a".equals("a")` → true，同时 `"a".hashCode() == "a".hashCode()` → true ✅
- `Integer(42).equals(Integer(42))` → true，同时 `hashCode` 都是 42 ✅

这就满足了 HashMap 的 key 的"金标准"——**equals 为 true 时 hashCode 一定相同**。

---

### 原因三：额外加分项——Final 类 + 实现了 Comparable

**1. String 和 Integer 都是 final 类，不能被继承**

这意味着你不用担心有人继承它们重写 hashCode/equals 搞出奇怪的问题。它们的行为是**确定的、可预期的**。

**2. 实现了 Comparable 接口**

String 和 Integer 都实现了 `Comparable`，这意味着它们不仅适合 HashMap，也适合 TreeMap：

```java
// 同一个 key 类型，HashMap 和 TreeMap 都能用
Map<String, Integer> hashMap = new HashMap<>();   // ✅
Map<String, Integer> treeMap = new TreeMap<>();   // ✅ String 自带比较逻辑

// 自定义类如果没有实现 Comparable，TreeMap 就用不了（除非传 Comparator）
Map<Student, Integer> treeMap2 = new TreeMap<>(); // ❌ ClassCastException
```

---

### 对比表：String vs 自定义类

| 对比项 | String | 自定义类 |
|:---|:---|:---|
| 不可变性 | ✅ 不可变，hashCode 稳定 | 通常可变，需自己注意 |
| hashCode | ✅ 正确重写，分布均匀 | 需自己写，容易出错 |
| equals | ✅ 正确重写，按值比较 | 需自己写，容易出错 |
| 一致性 | ✅ equals = true 时 hashCode 必相同 | 需自己保证 |
| 是否 final | ✅ final，行为确定 | 可能被继承覆盖 |
| 实现 Comparable | ✅ 已实现 | 通常没有 |

### 面试回答

> **面试官：为什么 HashMap 中 String、Integer 这样的包装类适合作为 key？**
>
> 主要有三个原因。
>
> 第一，**不可变性**。String 和 Integer 一旦创建就不会改变，所以它们的 hashCode 是稳定的。放入 HashMap 后 hashCode 不会变，保证任何时候 get 都能定位到同一个桶，不会出现"放进去就找不到了"的问题。
>
> 第二，**hashCode 和 equals 已经正确重写**。String 用 31 做乘数的散列算法分布均匀，Integer 的 hashCode 直接是自己的值，效率极高。两者的 equals 都是按值比较，并且 hashCode 和 equals 保持一致——equals 为 true 时 hashCode 一定相同。自己写的类很容易在这上面出错。
>
> 第三，**final 类且实现了 Comparable**。它们不能被继承，行为确定可靠。同时实现了 Comparable 接口，不仅可以用作 HashMap 的 key，也可以直接用作 TreeMap 的 key，通用性更强。

### 面试简答版

- **不可变性** → hashCode 稳定 → put 进去后不会找不到
- **已正确重写 hashCode+equals** → 拿来就用，不会踩坑
- **final 类** → 行为确定，不能被篡改
- **实现 Comparable** → HashMap 和 TreeMap 通用
- **对比自定义类**：自己写的类容易犯"忘了重写"或"写错了"的错误

## 13. 如果使用自定义 Object 作为 HashMap 的 key，应该怎么办？

### 先讲场景

前两题说了 String、Integer 适合做 key 是因为它们"不可变 + 已正确重写 hashCode/equals"。但如果业务需要，你必须拿自己的对象做 key，比如用 `Student` 类根据学号查分数——这时候该怎么做？

**核心就两件事：把 hashCode 和 equals 写对，然后别改 key。**

---

### 第一步：重写 equals

```java
@Override
public boolean equals(Object o) {
    // 1. 同一个引用 → true
    if (this == o) return true;
    
    // 2. 类型不对 → false
    if (o == null || getClass() != o.getClass()) return false;
    
    // 3. 转型后比较关键字段
    Student that = (Student) o;
    return Objects.equals(this.id, that.id);
}
```

**equals 的五个规定**（来自 Java 官方文档）：

```
自反性：x.equals(x) → true
对称性：x.equals(y) = y.equals(x)
传递性：x.equals(y) 且 y.equals(z) → x.equals(z)
一致性：只要字段不变，多次调用结果相同
非空性：x.equals(null) → false
```

---

### 第二步：重写 hashCode

**最重要的规则：equals 中用到的字段，必须全部参与 hashCode 的计算。**

```java
// ✅ 正确：equals 用了 id，hashCode 也用 id
@Override
public int hashCode() {
    return Objects.hash(id);
}

// ❌ 错误：equals 用了 id，但 hashCode 只用 name
@Override
public int hashCode() {
    return Objects.hash(name);  // 这样 id 相同但 name 不同的两个对象
}                               // equals 为 true，但 hashCode 不同 → 违反约定
```

**推荐的 hashCode 写法——用 `Objects.hash()`：**

```java
// 单个字段
@Override
public int hashCode() {
    return Objects.hash(id);
}

// 多个字段（用逗号分隔）
@Override
public int hashCode() {
    return Objects.hash(id, name, age);
}
```

`Objects.hash()` 内部帮你做了质数 31 的乘积计算，写法简单，不容易出错。

**如果不想用 Objects.hash（追求性能）：**

```java
@Override
public int hashCode() {
    int result = 1;
    result = 31 * result + (id == null ? 0 : id.hashCode());
    result = 31 * result + (name == null ? 0 : name.hashCode());
    return result;
}
```
两种写法等价，`Objects.hash()` 底层也是 31 质数法。

---

### 第三步：保证不可变（或至少别改 key）

```java
// 方案 A：把类做成不可变的（推荐）
public final class Student {    // final 防止继承
    private final String id;    // final 防止重新赋值
    private final String name;
    
    // 只有构造器，没有 setter
    public Student(String id, String name) {
        this.id = id;
        this.name = name;
    }
    // getter 可以有，setter 不要有
    public String getId() { return id; }
    public String getName() { return name; }
    
    @Override
    public boolean equals(Object o) { ... }
    @Override
    public int hashCode() { return Objects.hash(id); }
}
```

```java
// 用法：放心用，永远不会变
Student key = new Student("001", "张三");
map.put(key, 90);

// 没有 setter 方法，想改也改不了 ✅
// 任何时候 get 都能找到
```

**如果做不到不可变：**

```java
// 方案 B：对象可变，但约定"放进去后别改"
class Student {
    private String id;   // 有 setter
    private String name;
    
    public void setId(String id) { this.id = id; }
    // ...
}

Student key = new Student("001", "张三");
map.put(key, 90);

key.setId("002");  // ⚠️ 千万别！hashCode 变了，再也找不到了
```

```java
// 方案 C：用不可变字段做 key 的依据
// 比如 id 在构造后永远不变（final），name 可以变
class Student {
    private final String id;   // final！构造后就定死了
    private String name;        // 这个可以变
    
    @Override
    public int hashCode() {
        return Objects.hash(id);  // 只用不可变的 id 算 hashCode
    }
    @Override
    public boolean equals(Object o) {
        // 也只比 id
        return o instanceof Student s && Objects.equals(this.id, s.id);
    }
}
```
这样即使 name 变了，hashCode 也不受影响。

---

### 完整示例：一个合格的 key 类

```java
public final class Student {
    private final String id;
    private final String name;
    
    public Student(String id, String name) {
        this.id = id;
        this.name = name;
    }
    
    public String getId() { return id; }
    public String getName() { return name; }
    
    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Student s)) return false;
        return Objects.equals(id, s.id);
    }
    
    @Override
    public int hashCode() {
        return Objects.hash(id);
    }
    
    @Override
    public String toString() {
        return "Student{id='" + id + "', name='" + name + "'}";
    }
}
```

---

### 面试追问

**问：为什么 equals 和 hashCode 要用相同的字段？**

因为 HashMap 的约定是：**equals 为 true → hashCode 必须相同。** 如果 equals 用 id 判断，但 hashCode 用 name 算，两个 id 相同 name 不同的对象 equals 为 true，但 hashCode 不同，HashMap 会把他们放到不同的桶里——`get` 就找不到了。

**问：为什么推荐用 `Objects.hash()`？**

省事、防错。手写 31 质数法容易写错（忘了判空、乘错了数等）。`Objects.hash()` 底层已经帮你处理好了，一行搞定。

**问：key 类一定要是不可变的吗？**

不一定，但强烈推荐。如果对象可变，你在放进去之后不小心改了它的 key 字段，就永远取不出来了。如果实在做不到不可变，至少保证参与 hashCode/equals 的字段是 final 的。

### 面试回答

> **面试官：如果使用自定义对象作为 HashMap 的 key，应该怎么办？**
>
> 要做三件事。
>
> 第一，重写 equals，遵循 Java 官方的五条规定（自反、对称、传递、一致、非空）。equals 中用到的字段，必须是能唯一标识这个对象的业务字段，比如学生的学号。
>
> 第二，重写 hashCode，并且一定要保证和 equals 使用相同的字段。推荐用 `Objects.hash()` 一行搞定，不容易出错。这条约定来自于：如果两个对象 equals 为 true，它们的 hashCode 必须相同，否则 HashMap 会把它们放到不同的桶里。
>
> 第三，尽量让类不可变——字段用 private final，不提供 setter。如果做不到，至少保证参与 hashCode/equals 的字段不会被修改。如果放进去之后修改了 key 对象，hashCode 变了，就永远找不到了。

### 面试简答版

- **规则**：equals 和 hashCode 必须用**相同的字段**
- **equals 写法**：先判引用 → 再判类型 → 最后比字段（用 `Objects.equals`）
- **hashCode 写法**：用 `Objects.hash(字段1, 字段2, ...)` 一行搞定
- **推荐不可变**：字段加 `final`，不提供 setter
- **做不到不可变**：至少确保参与 hashCode 的字段是 final 的
- **⚠️ 绝对不要**：放进去之后修改 hashCode 相关的字段

## 14. HashMap 为什么不直接使用 hashCode() 处理后的哈希值直接作为 table 的下标？

### 先讲场景

```java
map.put("张三", 90);
"张三".hashCode();  // 比如 774573
```

你看，`hashCode()` 算出来是 774573，数组才 16 个格子（默认）。**总不能直接用 774573 当数组下标吧？数组没这么大。**

所以肯定要把它映射到数组长度范围内。但问题是：**映射完之后，为什么还要多做一步扰动（`h ^ (h >>> 16)`）？**

原因有两个，层层递进。

---

### 原因一（显而易见）：hashCode 范围太大，数组装不下

```java
// hashCode 的范围
int 的取值范围：-2^31 ~ 2^31-1（约 ±21 亿）

// 默认 HashMap 数组大小
默认初始容量：16

// 你不可能用一个 21 亿的数字给只有 16 个格子的数组当下标
// 所以必须映射——用 (n - 1) & hash 把它压缩到 0~15 之间
```

这就像你有一个 21 亿人的电话号码本，但只有 16 个抽屉来放——你必须通过某种方式决定谁放哪个抽屉。

---

### 原因二（核心）：直接取模会导致"高位失效"

假设我们不做扰动，直接拿 hashCode 来算下标：

```java
// 数组长度 16
int index = hashCode & 15;  // 15 = 1111（二进制）
// 这个运算只取了 hashCode 的低 4 位
// hashCode 的高 28 位全部被丢弃了！
```

画个图看：

```
hashCode（32 位二进制）：
1101 0101 0110 0111 1001 0011 1010 0101
↑                             ↑
高 28 位（被丢弃）             低 4 位（唯一决定下标）
                              1010 = 10 → 下标 10
```

**高 28 位的信息完全没有参与下标计算**。这意味着：如果两个对象的 hashCode 低位相同但高位不同，它们就一定会冲突。

举个例子：

```java
// 两个对象的 hashCode 完全不同（高位不同）
对象 A：hashCode = 1101 0101 0110 0111 1001 0011 1010 0101  → 下标 = 0101 = 5
对象 B：hashCode = 0010 1000 1100 1001 0111 1000 1010 0101  → 下标 = 0101 = 5 ！冲突

// 虽然 A 和 B 的 hashCode 完全不同（高位差异很大）
// 但低 4 位都是 0101 → 都落在下标 5 → 白冲突了
```

**这种情况很常见**——比如两个对象的 hashCode 虽然分布均匀，但取模后正好低位相同，就白白冲突了。更糟糕的是，如果某些类的 hashCode 实现不好（低位重复率高），冲突会更严重。

---

### 扰动函数怎么解决？——把高位信息"掺进"低位

```java
// JDK 8 的扰动函数
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

做的事情很简单：**把 hashCode 的高 16 位和低 16 位做异或，让低位混入高位的信息。**

```
原始 hashCode：     1101 0101 0110 0111 | 1001 0011 1010 0101
                  ↑ 高 16 位            ↑ 低 16 位

右移 16 位：        0000 0000 0000 0000 | 1101 0101 0110 0111

异或：              1101 0101 0110 0111 | 0100 0110 1100 0010
                                       ↑ 现在低 16 位里混入了高位的信息
```

扰动之后再取低 4 位，就不只是原来的低 4 位了——**它同时受到了高位的影响**：

```
扰动前：低 4 位 = 0101（仅取决于原始低 4 位）
扰动后：低 4 位 = 0010（混入了高位信息，分布更随机）
```

回到刚才的例子：

```java
// 不做扰动
对象 A：hashCode 低 4 位 = 0101 → 下标 5
对象 B：hashCode 低 4 位 = 0101 → 下标 5  ❌ 冲突

// 做扰动之后
对象 A：hash ^ (hash >>> 16) → 低 4 位变成 0010 → 下标 2
对象 B：hash ^ (hash >>> 16) → 低 4 位变成 1001 → 下标 9  ✅ 不冲突了
```

---

### 扰动程度对比：JDK 7 vs JDK 8

```java
// JDK 7：扰动 4 次——当年没有红黑树兜底，所以扰动要做得足够彻底
static int hash(int h) {
    h ^= (h >>> 20) ^ (h >>> 12);
    return h ^ (h >>> 7) ^ (h >>> 4);
}

// JDK 8：扰动 1 次——有红黑树兜底，即使冲突严重也有树保底
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

JDK 8 的开发者认为：有红黑树兜底（链表 ≥ 8 转树），就算扰动不够充分导致冲突多点，性能也不会崩。所以只做一次扰动就够了，简单高效。

---

### 完整流程串起来

```
key（"张三"）
    │
    ▼
key.hashCode()    → 774573（原始 hashCode，范围 ±21 亿）
    │
    ▼
h ^ (h >>> 16)    → 扰动后的 hash 值（高位信息掺进低位）
    │
    ▼
hash & (n - 1)    → 数组下标（映射到 0~15 之间）
    │
    ▼
table[index]      → 找到桶，放进去
```

**所以回答题目的问题：为什么不直接用 hashCode？**

```
① hashCode 范围太大（±21 亿），数组装不下 → 必须用 & 运算映射
② 即使映射后，如果 hashCode 只用低位 → 高位信息浪费，冲突增多
③ 所以加一步扰动 → 让高位也参与下标计算 → 分布更均匀 → 减少冲突
```

### 面试回答

> **面试官：HashMap 为什么不直接使用 hashCode() 作为 table 的下标？**
>
> 有两个原因。
>
> 第一，hashCode 的范围太大，一个 int 约 ±21 亿，而 HashMap 的数组默认才 16 个格子。不可能直接用 hashCode 当数组下标，必须映射到数组长度范围内。
>
> 第二，即使通过 `(n - 1) & hash` 做了映射，如果不做扰动，计算下标时只用到了 hashCode 的低位，高位的信息全部浪费了。这会导致本来 hashCode 差异很大的两个 key，因为低位相同而发生冲突。
>
> 所以 HashMap 在取模之前加了一步扰动：`h ^ (h >>> 16)`，把高 16 位的信息异或到低 16 位上。这样即使数组很小（只看低几位），高位也能影响下标的计算结果，哈希分布更均匀，冲突更少。JDK 8 的扰动只做一次，因为有了红黑树兜底，不需要像 JDK 7 那样做四次。

### 面试简答版

- **原因 ①**：hashCode 范围太大（±21 亿），数组装不下 → 必须映射
- **原因 ②**：直接 `& (n-1)` 只用到低位，高位信息浪费 → 冲突增加
- **解决方案**：扰动函数 `h ^ (h >>> 16)` → 高位信息掺进低位 → 分布更均匀
- **JDK 7 vs 8**：7 是四次扰动，8 是一次扰动（有红黑树兜底，不用太复杂）
- **完整链路**：key → hashCode → 扰动 → `& (n-1)` → 数组下标

## 15. HashMap 的长度为什么是 2 的幂次方？

### 先讲场景

你有没有想过，为什么 HashMap 的默认容量是 16 而不是 10？为什么扩容是翻倍而不是加一半？为什么你传 `new HashMap(15)`，它自动给你变成 16？

```java
// 你传的不是 2 的幂
Map<String, String> map = new HashMap<>(15);
// 实际上内部变成了 16

Map<String, String> map2 = new HashMap<>(17);
// 实际上内部变成了 32
```

**所有答案都指向同一个原因：当容量是 2 的幂时，`(n - 1) & hash` 才能发挥最大作用。** 具体来说有三个好处。

---

### 好处一：用位运算代替取模，性能更高

```java
// 计算下标的公式
int index = (n - 1) & hash;
```

如果 n 是 2 的幂，`(n - 1) & hash` **完全等价于** `hash % n`，但 `&` 运算只要一个 CPU 周期，而 `%` 需要几十个周期。

```java
n = 16（2⁴）：n - 1 = 15 = 1111（二进制）
hash & 1111 = 取 hash 的低 4 位 → 范围 0~15

这完全等价于 hash % 16，但 & 运算快得多
```

这就是 HashMap 追求的核心——**每次 put/get 都要算一次下标，用 `&` 替代 `%` 能省下可观的性能。**

---

### 好处二：下标分布均匀，减少冲突

这是最直观的理解。看 n 不是 2 的幂会发生什么：

```java
// n = 15（不是 2 的幂）
n - 1 = 14 = 1110（二进制）

// hash & 14 的结果：最低位永远是 0
// → 奇数下标（1, 3, 5, 7, 9, 11, 13）永远不会被使用
// → 只有 0, 2, 4, 6, 8, 10, 12, 14 这 8 个位置可用
// → 少了一半的空间，冲突概率翻倍
```

```
数组长度 15：                      数组长度 16：
┌────┬────┬────┬────┬────┐        ┌────┬────┬────┬────┬────┬────┐
│ 0  │ 2  │ 4  │ 6  │ 8  │        │ 0  │ 1  │ 2  │ 3  │ 4  │ 5  │
│ unused │ unused │      │        │    │    │    │    │    │    │
│ 10  │ 12 │ 14 │    │    │        │ 6  │ 7  │ 8  │ 9  │10  │11  │
└────┴────┴────┴────┴────┘        │    │    │    │    │    │    │
 浪费了一半位置 ❌                  │12  │13  │14  │15  │    │    │
                                   └────┴────┴────┴────┴────┴────┘
                                    所有位置都能用 ✅
```

**关键区别：**

```
n 是 2 的幂时，n - 1 的二进制全是 1：
  16 - 1 = 1111（4 个 1）→ hash 的任意位组合都能映射到 0~15
  → 每个位置概率均等，分布均匀 ✅

n 不是 2 的幂时，n - 1 的二进制含有 0：
  15 - 1 = 1110（最低位是 0）→ hash 的最低位被屏蔽
  → 奇数位置永远用不上 → 空间浪费，冲突翻倍 ❌
```

### 好处三：扩容时元素能均匀拆分，且不用 rehash

这是 JDK 8 最精妙的设计，和第 9 题扩容的内容直接相关。

```java
// 扩容时的判断：元素留在原位，还是挪到"原位 + 旧容量"
if ((e.hash & oldCap) == 0) {
    // 留在原位
} else {
    // 挪到 原位 + oldCap
}
```

这个判断之所以成立，**前提就是旧容量是 2 的幂**。

```
旧容量 16（10000），新容量 32（100000）

扩容后多看了 hash 的第 5 位：
  - 第 5 位 = 0 → 原位
  - 第 5 位 = 1 → 原位 + 16

可以看到：新增的那一位把原桶里的元素均匀地分成了两半
一半留在原位，一半挪到高位
```

如果容量不是 2 的幂，扩容时就做不到这种"均匀拆分"——要么需要重新计算每个元素的下标，要么拆分不均匀。

这个机制在 [[06Map接口.md#9 HashMap 的扩容操作是怎么实现的？|第 9 题扩容]] 里详细讲过。

---

### HashMap 怎么保证容量一定是 2 的幂？

如果你传的不是 2 的幂，HashMap 会自动帮你"修正"：

```java
// HashMap 的 tableSizeFor 方法
// 把你传的容量，调整成最接近的 2 的幂
static final int tableSizeFor(int cap) {
    int n = cap - 1;
    n |= n >>> 1;
    n |= n >>> 2;
    n |= n >>> 4;
    n |= n >>> 8;
    n |= n >>> 16;
    return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
}
```

```
你传 15：  经过计算 → 返回 16 ✅
你传 17：  经过计算 → 返回 32 ✅
你传 100：经过计算 → 返回 128 ✅
你传 1024：本身就是 2 的幂 → 返回 1024 ✅
```

而且扩容时 `newCap = oldCap << 1`，左移一位就是乘 2——**始终保持 2 的幂**。

---

### 总结

| 好处 | 说明 |
|:---|:---|
| **性能** | `&` 位运算替代 `%` 取模，一个 CPU 周期 vs 几十个周期 |
| **均匀** | `(n-1)` 二进制全 1，每个位置概率均等，冲突少 |
| **扩容** | `e.hash & oldCap` 判断原位/挪位，拆分均匀，无需 rehash |

### 面试回答

> **面试官：HashMap 的长度为什么是 2 的幂次方？**
>
> 主要有三个原因。
>
> 第一，性能。用 `(n - 1) & hash` 替代 `hash % n` 来计算下标，位运算比取模快得多，而每次 put/get 都要算一次下标。
>
> 第二，分布均匀。当 n 是 2 的幂时，n - 1 的二进制全是 1，hash 的任何位都有机会参与下标计算，每个位置被选中的概率均等。如果 n 不是 2 的幂，n - 1 的二进制中含有 0，会导致某些位置永远用不上，空间浪费一半，冲突概率翻倍。
>
> 第三，扩容优化。扩容时用 `e.hash & oldCap` 判断元素是留在原位还是挪到"原位 + 旧容量"，新增的那一位能把原桶里的元素均匀分成两半。这个设计的前提就是容量必须是 2 的幂。
>
> 如果用户传的容量不是 2 的幂，HashMap 内部会通过 tableSizeFor 方法自动调整成最接近的 2 的幂。

### 面试简答版

- **性能**：`&` 位运算代替 `%` 取模，更快
- **均匀**：`(n-1)` 二进制全 1，下标分布均匀，不浪费空间
- **扩容**：`e.hash & oldCap` 均匀拆分，无需 rehash
- **自动修正**：你传非 2 的幂，`tableSizeFor()` 帮你调成最近的 2 的幂

## 16. HashMap 与 Hashtable 有什么区别？

### 先讲场景

Hashtable 是 JDK 1.0 就存在的"老古董"，HashMap 是 JDK 1.2 随着集合框架一起推出的"新秀"。两者都是 Map 接口的实现，但 Hashtable 基本已经被淘汰了——**面试问它的区别，主要是为了让你理解为什么 Hashtable 不好用，以及 ConcurrentHashMap 是怎么取代它的。**

主要区别有六点。

---

### 区别一：线程安全

```java
// Hashtable：方法用 synchronized 修饰，线程安全但性能差
public synchronized V put(K key, V value) { ... }
public synchronized V get(Object key) { ... }

// HashMap：线程不安全，但性能好
public V put(K key, V value) { ... }
public V get(Object key) { ... }
```

Hashtable 每个方法都加了 `synchronized`，虽然保证了线程安全，但**锁的粒度太粗**——不管读还是写，都锁整个哈希表。多线程环境下，一个线程在 put，其他所有线程都得等着。

而 HashMap 本身不是线程安全的，想要线程安全应该用 ConcurrentHashMap，它只锁单个桶，并发性能远高于 Hashtable。

对比一下三种 Map 的线程安全方案：

| Map | 线程安全 | 锁粒度 | 性能 |
|:---|:---|:---|:---|
| **HashMap** | ❌ 不安全 | 无锁 | 最高 |
| **Hashtable** | ✅ 安全 | **整张表**（粗锁） | 低 |
| **ConcurrentHashMap** | ✅ 安全 | 单个桶（细锁） | 高 |

详细见 [[03集合容器概述.md#8 哪些集合类是线程安全的？]] 的 ConcurrentHashMap 部分。

---

### 区别二：null 值处理

```java
// HashMap：允许 null key 和 null value
Map<String, String> map = new HashMap<>();
map.put(null, "value");   // ✅ null key 放在 table[0]
map.put("key", null);     // ✅ null value 允许
map.get(null);            // "value"

// Hashtable：不允许 null
Map<String, String> table = new Hashtable<>();
table.put(null, "value");  // ❌ NullPointerException
table.put("key", null);   // ❌ NullPointerException
```

为什么 Hashtable 不允许 null？

```java
// Hashtable 的 put 方法
public synchronized V put(K key, V value) {
    if (value == null) throw new NullPointerException();  // 直接抛异常
    // ...
    int hash = key.hashCode();  // key 为 null 时 → NullPointerException
    // ...
}
```

HashMap 则做了特殊处理：

```java
// HashMap 的 put 方法
static final int hash(Object key) {
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    // key 为 null → hash = 0 → 放到 table[0]
}
```

---

### 区别三：初始容量和扩容方式

```java
// HashMap
默认初始容量：16（2⁴）
扩容方式：    newCap = oldCap << 1（翻倍）
容量必须是 2 的幂

// Hashtable
默认初始容量：11
扩容方式：    newCapacity = oldCapacity * 2 + 1（翻倍+1）
容量没有强制要求
```

HashMap 的容量设计（2 的幂）是为了：
- `&` 运算替代 `%` 提升性能
- 下标分布均匀
- 扩容时 `e.hash & oldCap` 均匀拆分

这些在 [[06Map接口.md#15 HashMap 的长度为什么是 2 的幂次方？|第 15 题]] 详细讲过。

Hashtable 用 `* 2 + 1` 是想让容量保持奇数，减少哈希冲突——但这是个旧时代的做法，效果不如 HashMap 的 2 的幂设计。

---

### 区别四：底层数据结构和 hash 算法

```java
// HashMap：数组 + 链表 + 红黑树（JDK 8+）
// 有扰动函数
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}

// Hashtable：数组 + 链表（永远只有链表，没有红黑树）
// 直接用 hashCode，没有扰动
int hash = key.hashCode();
int index = (hash & 0x7FFFFFFF) % table.length;
```

| 对比 | HashMap（JDK 8） | Hashtable |
|:---|:---|:---|
| 数据结构 | 数组 + 链表 + **红黑树** | 数组 + 链表（永不转树） |
| hash 算法 | `h ^ (h >>> 16)` 扰动 | 直接用 `hashCode` |
| 下标计算 | `(n - 1) & hash` | `(hash & 0x7FFFFFFF) % len` |

Hashtable 用 `& 0x7FFFFFFF` 是为了把负数 hash 变成正数（`hashCode` 可能是负数），然后再取模。而 HashMap 的 `&` 运算结果自动是非负的，不需要这一步。

---

### 区别五：遍历方式

```java
// HashMap：Iterator（fail-fast）
Map<String, String> map = new HashMap<>();
Iterator<Map.Entry<String, String>> iter = map.entrySet().iterator();

// Hashtable：有 Enumerator（非 fail-fast）和 Iterator（fail-fast）
Enumeration<String> keys = table.keys();  // 旧式遍历
Iterator<String> iter2 = table.keySet().iterator();  // 新式遍历
```

HashMap 使用 Iterator，遵循 fail-fast 机制——遍历时如果有其他线程修改结构，会抛 `ConcurrentModificationException`。

Hashtable 保留了两种遍历方式：
- `Enumeration`（来自 JDK 1.0）——**不抛异常**，也不保证遍历结果正确
- `Iterator`——也支持 fail-fast

---

### 区别六：继承的父类

```java
// HashMap 继承 AbstractMap
public class HashMap<K,V> extends AbstractMap<K,V>

// Hashtable 继承 Dictionary（已废弃）
public class Hashtable<K,V> extends Dictionary<K,V>
```

`Dictionary` 是 JDK 1.0 的抽象类，早在 JDK 1.2 就被标记为废弃了。`AbstractMap` 是集合框架里的标准实现，提供了很多默认方法。

---

### 总结对比表

| 对比项 | HashMap | Hashtable |
|:---|:---|:---|
| **线程安全** | ❌ 不安全 | ✅ 安全（synchronized，但粗锁性能差） |
| **null key** | ✅ 允许 | ❌ 抛 NullPointerException |
| **null value** | ✅ 允许 | ❌ 抛 NullPointerException |
| **默认容量** | 16 | 11 |
| **扩容方式** | `oldCap << 1`（翻倍） | `old * 2 + 1`（翻倍+1） |
| **数据结构** | 数组+链表+红黑树 | 数组+链表 |
| **hash 算法** | 扰动后 `& (n-1)` | 去符号后 `% len` |
| **遍历方式** | Iterator（fail-fast） | Enumerator + Iterator |
| **父类** | AbstractMap | Dictionary（已废弃） |
| **诞生时间** | JDK 1.2 | JDK 1.0 |
| **推荐使用** | ✅ 单线程首选 | ❌ 已淘汰，用 ConcurrentHashMap 替代 |

### 面试回答

> **面试官：HashMap 和 Hashtable 有什么区别？**
>
> 主要有六个区别。
>
> 第一，线程安全。Hashtable 的方法用 synchronized 修饰，是线程安全的，但锁整张表性能差。HashMap 不是线程安全的。并发场景应该用 ConcurrentHashMap。
>
> 第二，null 值。HashMap 允许一个 null key 和多个 null value。Hashtable 不允许 null，key 或 value 为 null 就抛 NullPointerException。
>
> 第三，初始容量和扩容。HashMap 默认 16，每次翻倍，容量始终是 2 的幂。Hashtable 默认 11，每次翻倍再加 1。
>
> 第四，数据结构。HashMap JDK 8 引入了红黑树优化长链表。Hashtable 只有链表，没有这个优化。
>
> 第五，hash 算法。HashMap 有扰动函数，把高位和低位异或后再计算下标。Hashtable 直接用 hashCode 取模。
>
> 第六，Hashtable 是 JDK 1.0 的遗留类，继承自废弃的 Dictionary 类，现在已经被 ConcurrentHashMap 取代了。

### 面试简答版

- **线程安全**：Hashtable 安全但粗锁，HashMap 不安全
- **null**：HashMap 允许 null key/value，Hashtable 不允许
- **容量**：HashMap 默认 16 扩容翻倍，Hashtable 默认 11 扩容翻倍+1
- **数据结构**：HashMap 有红黑树优化，Hashtable 只有链表
- **hash 算法**：HashMap 有扰动，Hashtable 直接用 hashCode 取模
- **结论**：**Hashtable 已淘汰**，单线程用 HashMap，多线程用 ConcurrentHashMap

## 17. 什么是 TreeMap？

### 先讲场景

HashMap 是无序的——它不保证元素的顺序，遍历顺序可能每次都不一样。但有时候你需要 Map 的 key **按自然顺序排列**，比如按学号从小到大遍历、按时间从小到大取数据。这时候就用 TreeMap。

**一句话：TreeMap 是基于红黑树的、key 自动排序的 Map。**

```java
Map<String, String> map = new TreeMap<>();
map.put("B", "2");
map.put("A", "1");
map.put("C", "3");

System.out.println(map);  // {A=1, B=2, C=3} ← 自动按字母排序了 ✅
```

---

### 核心特性

| 特性 | 说明 |
|:---|:---|
| **底层结构** | **红黑树**（自平衡二叉查找树） |
| **顺序** | key 按自然顺序或 Comparator 排序 |
| **时间复杂度** | put / get / remove 都是 **O(log n)** |
| **线程安全** | ❌ 不安全 |
| **null key** | ⚠️ 不允许（自然排序时），除非 Comparator 支持 |
| **null value** | ✅ 允许 |

和 HashMap 的核心差异：

```
HashMap：  数组 + 链表/红黑树  → O(1) → 无序
TreeMap：  红黑树            → O(log n) → 有序
```

---

### 两种排序方式

#### 1. 自然排序（key 必须实现 Comparable）

```java
// String、Integer 都实现了 Comparable，可以直接用
TreeMap<String, Integer> map1 = new TreeMap<>();   // ✅ String 自然排序
TreeMap<Integer, String> map2 = new TreeMap<>();   // ✅ Integer 自然排序

// 自定义类没有实现 Comparable → 抛异常 ❌
TreeMap<Student, String> map3 = new TreeMap<>();   // ClassCastException
```

#### 2. 指定 Comparator

```java
// 按字符串长度排序
TreeMap<String, String> map = new TreeMap<>((a, b) -> a.length() - b.length());
map.put("aaa", "3");
map.put("b", "1");
map.put("cc", "2");
System.out.println(map);  // {b=1, cc=2, aaa=3} ← 按长度排序了

// 自定义类传 Comparator 也能用
TreeMap<Student, Integer> map2 = new TreeMap<>(
    (s1, s2) -> s1.getId().compareTo(s2.getId())
);  // ✅ 传了 Comparator，可以用了
```

---

### NavigableMap 的导航方法

TreeMap 实现了 `NavigableMap` 接口，提供了很多方便的导航方法：

```java
TreeMap<Integer, String> map = new TreeMap<>();
map.put(1, "A");
map.put(3, "C");
map.put(5, "E");
map.put(7, "G");
map.put(9, "I");

// 导航方法
map.lowerKey(5);     // 3    ← 比 5 小的最大 key
map.floorKey(5);     // 5    ← ≤ 5 的最大 key
map.ceilingKey(5);   // 5    ← ≥ 5 的最小 key
map.higherKey(5);    // 7    ← 比 5 大的最小 key

map.firstKey();      // 1    ← 最小的 key
map.lastKey();       // 9    ← 最大的 key

// 取子集
map.subMap(3, 7);    // {3=C, 5=E} ← key 在 [3, 7) 范围内的元素
map.headMap(5);      // {1=A, 3=C} ← key < 5 的元素
map.tailMap(5);      // {5=E, 7=G, 9=I} ← key ≥ 5 的元素
```

这些方法在处理区间查询（比如按时间范围取数据）时非常有用。

---

### TreeMap vs HashMap 怎么选？

| 需求 | 选哪个 |
|:---|:---|
| 只根据 key 查 value，不关心顺序 | **HashMap**（O(1) 更快） |
| 需要 key 按顺序遍历 | **TreeMap** |
| 需要 range 查询（取前 10、区间子集） | **TreeMap** |
| key 是自定义类，没实现 Comparable | HashMap（重写 hashCode/equals） |

```java
// 什么时候用 TreeMap？
// 1. 需要排序展示
Map<String, Integer> sortedMap = new TreeMap<>(data);
// 遍历时自动按 key 排序

// 2. 需要按范围取数据
TreeMap<Long, Event> timeline = new TreeMap<>();
// 取某段时间内的事件
timeline.subMap(startTime, endTime);
```

### 面试回答

> **面试官：什么是 TreeMap？**
>
> TreeMap 是基于红黑树实现的有序 Map。key 会自动排序——要么按自然顺序（key 实现 Comparable），要么按传入的 Comparator 排序。put、get、remove 的时间复杂度都是 O(log n)。
>
> 和 HashMap 比，TreeMap 的优势在于有序性：遍历时 key 是排好序的，还提供了 lowerKey、subMap 等导航方法，适合做区间查询和排序展示。缺点是性能不如 HashMap（O(log n) vs O(1)）。
>
> TreeMap 不允许 null key（因为 null 无法比较大小），不是线程安全的。

### 面试简答版

- **底层**：**红黑树**
- **顺序**：key 按 Comparable 或 Comparator 自动排序
- **性能**：put/get/remove 都是 **O(log n)**
- **导航方法**：lowerKey、subMap、firstKey 等区间查询
- **vs HashMap**：有序但慢一点，需要排序或区间查询时用 TreeMap
- **⚠️ 不允许 null key**（无法比较大小）

## 18. 如何决定使用 HashMap 还是 TreeMap？

### 先讲场景

上一题最后给了一个对比表，但实际开发中怎么选，很多人还是会纠结。比如：

- 我只是存一些数据，需要排序吗？不排序是不是就不用 TreeMap？
- 我只需要按插入顺序遍历，用哪个？
- 我用 TreeMap 会对性能有多大影响？

**核心决策逻辑就一句话：需要排序就用 TreeMap，否则用 HashMap。**

因为 HashMap 的 O(1) 性能在绝大多数场景下都优于 TreeMap 的 O(log n)。除非你明确需要 TreeMap 的有序性，否则默认选 HashMap 不会错。

---

### 场景一：不需要排序 → 无脑 HashMap

```java
// 最常见的场景：缓存数据、根据 ID 查询
Map<Integer, User> userCache = new HashMap<>();

userCache.put(1001, user1);
userCache.put(1002, user2);
// 我只靠 ID 查用户，不关心顺序

User u = userCache.get(1001);  // O(1) ✅ 快
```

HashMap 的 O(1) 和 TreeMap 的 O(log n) 在数据量大时差距明显：

```
数据量     HashMap     TreeMap
1 万        ~0.1ms      ~0.7ms    ← 10 万次查询差 6 秒
10 万       ~0.5ms      ~1.2ms
100 万      ~2ms        ~2.5ms    ← 差距缩小（JIT 优化后）
```

对于绝大多数业务场景，这个差距可以忽略。但**既然不用排序，为什么还要多付出 log n 的代价？**

---

### 场景二：需要排序 → TreeMap

```java
// 场景 A：按 key 顺序展示
Map<String, Integer> scores = new TreeMap<>();
scores.put("张三", 90);
scores.put("李四", 85);
scores.put("王五", 95);

// 遍历时自动按名字排序
for (Map.Entry<String, Integer> entry : scores.entrySet()) {
    System.out.println(entry.getKey() + " = " + entry.getValue());
}
// 李四=85, 张三=90, 王五=95  ← 按拼音字母排好了
```

```java
// 场景 B：区间查询（TreeMap 独有优势）
TreeMap<Long, String> logs = new TreeMap<>();
logs.put(100L, "启动");
logs.put(200L, "加载");
logs.put(300L, "渲染");
logs.put(400L, "完成");

// 取出 150~350 之间的日志
logs.subMap(150L, 350L);  // {200=加载, 300=渲染} ✅

// 取最大的 key
logs.lastKey();            // 400
```

这些**导航方法**（subMap、headMap、tailMap、lowerKey、ceilingKey 等）是 TreeMap 独有的，HashMap 做不到。

```java
// HashMap 做不到区间查询
Map<Long, String> map = new HashMap<>();
// 想取 150~350 之间的数据？只能遍历所有 key 一个个判断
```

---

### 场景三：只要按插入顺序遍历 → LinkedHashMap

如果你需要的不是"排序"，而只是"按放进去的顺序遍历"，**TreeMap 太重了**，用 LinkedHashMap 更合适：

```java
// LinkedHashMap：维护插入顺序，性能接近 HashMap
Map<String, Integer> map = new LinkedHashMap<>();
map.put("B", 2);
map.put("A", 1);
map.put("C", 3);

System.out.println(map);  // {B=2, A=1, C=3} ← 按插入顺序，不是按 key 排序
```

```
            性能          顺序
HashMap     O(1)         无序
LinkedHashMap O(1)       插入顺序（或访问顺序）
TreeMap     O(log n)     按 key 排序
```

如果你的需求只是"保持插入顺序"，LinkedHashMap 是比 TreeMap 更好的选择——既有 O(1) 的性能，又能保证顺序。

---

### 场景四：纠结"现在不排序，以后可能要排序"

```java
// 方案 A：先用 HashMap，需要排序时再转
Map<String, Integer> map = new HashMap<>();
// ... 存放数据 ...

// 需要排序时：
Map<String, Integer> sorted = new TreeMap<>(map);
// 一行转成 TreeMap
```

```java
// 方案 B：用 TreeSet 做排序索引
// 如果只是偶尔需要排序，维护两份数据可能更清晰
Map<String, User> userMap = new HashMap<>();               // 主存储
TreeSet<String> sortedKeys = new TreeSet<>(userMap.keySet()); // 排序索引

// 新增时同时维护：
userMap.put(id, user);
sortedKeys.add(id);
```

但大多数情况下，**方案 A 就够了**——先用 HashMap 爽着，真要排序时 new TreeMap<>(map) 一行搞定。

---

### 一张流程图帮你决策

```
要存 key-value 数据？
    │
    ├─ 需要 key 排序遍历？    → ✅ TreeMap
    │
    ├─ 需要区间查询？         → ✅ TreeMap
    │   （subMap、lowerKey 等）
    │
    ├─ 只需要插入顺序？       → ✅ LinkedHashMap
    │
    └─ 不关心顺序/只查 value  → ✅ HashMap
```

### 面试回答

> **面试官：如何决定使用 HashMap 还是 TreeMap？**
>
> 核心原则是：**需要排序就用 TreeMap，否则用 HashMap。**
>
> HashMap 的 O(1) 性能在绝大多数场景下优于 TreeMap 的 O(log n)，所以不需要排序时默认选 HashMap。
>
> TreeMap 的优势在于有序性——key 自动排序遍历、subMap 等导航方法做区间查询，这些是 HashMap 做不到的。
>
> 如果只是需要保持插入顺序，不是真的按 key 排序，可以用 LinkedHashMap 替代 TreeMap，它既有 O(1) 性能又能保证顺序。
>
> 如果拿不准未来要不要排序，先上 HashMap，需要排序时 `new TreeMap<>(hashMap)` 一行转换即可。

### 面试简答版

- **决策原则**：需要排序 → TreeMap，否则 → HashMap
- **HashMap**：O(1)，不关心顺序时的默认选择
- **TreeMap**：O(log n)，需要排序遍历或区间查询时用
- **LinkedHashMap**：O(1)，只要插入顺序时比 TreeMap 更合适
- **举棋不定**：先用 HashMap，需要排序时 `new TreeMap<>(map)` 一行转

## 19. HashMap 和 ConcurrentHashMap 的区别？

### 先讲场景

面试问这道题，其实不是要你背 HashMap 和 CHM 的所有区别，而是要你讲清楚**线程安全 Map 的演进路线**——从 Hashtable 到 JDK 7 CHM 到 JDK 8 CHM，锁粒度越来越细，性能越来越好的过程。

**核心就一句话：Hashtable 锁整张表 → 1.7 锁分段 → 1.8 锁单个桶，锁粒度越来越细。**

---

### 一、Hashtable——全局一把大锁

![[Pasted image 20260502214746.png]]
```java
public synchronized V put(K key, V value) { ... }  // 所有方法都加 synchronized
public synchronized V get(Object key) { ... }
```

```
一个线程 put/get，整个集合被锁住，其他线程全部阻塞 ❌
锁粒度太大，并发效率极低，基本淘汰
```


---

### 二、JDK 1.7 ConcurrentHashMap——分段锁（Segment）![[Pasted image 20260502214337.png]]

把大桶数组拆成多个小区域，每个区域一把锁：

```
JDK 1.7 ConcurrentHashMap：
┌──────────┬──────────┬──────────┬──────────┐
│ Segment0 │ Segment1 │ ......   │ Segment15│  ← 默认 16 个分段
│  ┌──┬──┐ │  ┌──┬──┐ │          │  ┌──┬──┐ │
│  │H0│H1│ │  │H0│H1│ │          │  │H0│H1│ │
│  └──┴──┘ │  └──┴──┘ │          │  └──┴──┘ │
└──────────┴──────────┴──────────┴──────────┘
  lock()      lock()                lock()    ← 每个 Segment 独享一把锁
```

```java
// 分段锁逻辑
put(key, value) {
    // 先定位到某个 Segment
    int segmentIndex = hash & (segments.length - 1);
    segments[segmentIndex].put(key, value);  // 只锁当前 Segment
    // 其他 15 个 Segment 不受影响 ✅
}
```

多个线程访问**不同分段**互不阻塞，只有访问**同一个分段**才竞争锁。并发度 = 默认 16。

---

### 三、JDK 1.8 ConcurrentHashMap——CAS + 锁单个桶
![[Pasted image 20260502214346.png]]
废弃 Segment，直接用和 HashMap 一样的底层结构（数组 + 链表 + 红黑树）。

```
JDK 1.8 ConcurrentHashMap：
┌────┬────┬────┬────┬────┬────┬────┬────┐
│ 0  │ 1  │ 2  │ 3  │ 4  │ 5  │ 6  │ 7  │  ← 直接操作 Node[]
└────┴────┴────┴────┴────┴────┴────┴────┘
 lock()  lock()              lock()          ← 每个桶的首节点单独锁
```

```java
// JDK 1.8 put 流程（伪代码）
final V putVal(K key, V value, ...) {
    // 1. 桶为空 → CAS 无锁插入
    if (tab[i] == null)
        casTabAt(tab, i, null, newNode);
    
    // 2. 桶不为空 → synchronized 锁首节点
    synchronized (tab[i]) {
        // 链表插入 / 红黑树插入
    }
}
```

**锁粒度再细化：**
- 不锁整张表（Hashtable）❌
- 不锁分段（1.7 CHM）
- **只锁单个桶的首节点**（1.8 CHM）✅

而且利用了 JDK 1.6 之后 **synchronized 锁升级优化**（偏向锁 → 轻量级锁 → 重量级锁），竞争不激烈时性能极好。

> 保留 Segment 只是为了版本兼容，实际已经不用了。

---

### 四、对比总结

**锁粒度演进：**
```
Hashtable       → 锁整张表（最粗，并发最差）❌
1.7 CHM         → 锁分段（16 个 Segment，中等）
1.8 CHM         → 锁单个桶首节点（最细，并发最好）✅
```

**HashMap vs CHM 其他区别：**

```java
// null 值
map.put(null, "value");           // HashMap ✅ 允许
concurrentMap.put(null, "value"); // CHM ❌ NullPointerException

// 迭代器
// HashMap:    fail-fast（遍历时结构变了就抛异常）
// CHM:        弱一致性（不抛异常，但不保证遍历到最新数据）
```

**底层结构（JDK 8）：**
| | HashMap | CHM |
|:---|:---|:---|
| 数据结构 | 数组+链表+红黑树 | 数组+链表+红黑树 |
| 并发控制 | 无锁 | CAS + synchronized |
| null key | ✅ | ❌ |
| 迭代器 | fail-fast | 弱一致性 |
| 使用场景 | 单线程 | 多线程 |

详细深入内容在 [[03集合容器概述.md#8 哪些集合类是线程安全的？]]。

### 面试回答

> **面试官：HashMap 和 ConcurrentHashMap 有什么区别？**
>
> 核心区别在于线程安全和锁粒度的演进。
>
> 最早是 Hashtable，所有方法加 synchronized，锁整张表，并发效率极低，基本淘汰。
>
> JDK 7 的 ConcurrentHashMap 引入分段锁（Segment），默认 16 个分段，每个分段单独一把锁。线程访问不同分段互不阻塞，并发度大幅提升。
>
> JDK 8 的 ConcurrentHashMap 废弃了 Segment，改用和 HashMap 一样的数组 + 链表 + 红黑树结构，并发控制改用 CAS + synchronized，锁粒度细化到单个桶的首节点。还利用了 synchronized 锁升级优化，性能更好。
>
> 其他区别：HashMap 允许 null key/value，CHM 不允许；HashMap 迭代器是 fail-fast，CHM 是弱一致性的。

### 面试简答版

- **Hashtable**：锁整张表，并发最差 ❌
- **1.7 CHM**：分段锁（默认 16 个 Segment），并发中等
- **1.8 CHM**：**CAS + 锁单个桶首节点**，并发最好 ✅
- **null**：HashMap 允许，CHM 不允许
- **迭代器**：HashMap fail-fast，CHM 弱一致性
- **底层**：JDK 8 两者基本一致（数组+链表+红黑树）

## 20. ConcurrentHashMap 的底层实现原理？

### JDK 1.7：Segment + HashEntry（分段锁）

首先将数据分为一段一段的存储，然后给每一段数据配一把锁。当一个线程占用锁访问其中一个段数据时，其他段的数据也能被其他线程访问。
![[Pasted image 20260502214840.png]]

```
ConcurrentHashMap（JDK 1.7）：
┌──────────────┬──────────────┬──────────────┬──────────────┐
│  Segment[0]  │  Segment[1]  │  ......      │  Segment[15] │  ← 默认 16 个 Segment
│ ┌──┬──┬──┐  │ ┌──┬──┬──┐  │              │ ┌──┬──┬──┐  │
│ │H0│H1│H2│  │ │H0│H1│  │  │              │ │H0│H1│H2│  │  ← HashEntry 数组
│ └──┴──┴──┘  │ └──┴──┴──┘  │              │ └──┴──┴──┘  │
└──────────────┴──────────────┴──────────────┴──────────────┘
      锁 ↑           锁 ↑                       锁 ↑        ← 每个 Segment 一把锁
```

结构：
- 该类包含两个静态内部类：**HashEntry**（封装键值对）和 **Segment**（充当锁的角色）
- Segment 是一种**可重入锁 ReentrantLock**，每个 Segment 守护一个 HashEntry 数组里的元素
- 当对 HashEntry 数组的数据进行修改时，必须先获得对应的 Segment 的锁
- 一个 ConcurrentHashMap 包含一个 **Segment 数组**，Segment 的结构和 HashMap 类似（数组 + 链表）

并发度 = Segment 数量（默认 16），理论上有 16 个线程可以同时写。

### JDK 8：CAS + synchronized（锁单个桶）
![[Pasted image 20260502215022.png]]

- 放弃 Segment 臃肿的设计，采用 **Node + CAS + Synchronized** 来保证并发安全
- synchronized 只锁定当前链表或红黑二叉树的首节点
- 只要 hash 不冲突，就不会产生并发，效率又提升 N 倍
- 底层结构和 HashMap 一样：**数组 + 链表 + 红黑树**
- 空桶插入用 CAS 无锁竞争，非空桶用 synchronized 锁头节点

### 详细源码分析

put、get、扩容 transfer、size 计数等完整源码分析在 [[03集合容器概述.md#8 哪些集合类是线程安全的？|03 集合容器概述#8]] 里。
