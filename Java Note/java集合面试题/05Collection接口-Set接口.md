# Set 接口

## 1. 说一下 HashSet 的实现原理？

### 先讲场景

你平时用 `HashSet` 的时候，有没有想过一个问题：

> HashSet 怎么做到"自动去重"的？它怎么知道两个元素是不是重复？

如果你看过源码，会发现一个惊人的事实——**HashSet 底层其实是 HashMap**。

```java
// HashSet 源码（JDK 8）
public class HashSet<E> {
    // 底层就是一个 HashMap
    private transient HashMap<E, Object> map;

    // 所有 value 都指向同一个固定对象
    private static final Object PRESENT = new Object();

    public HashSet() {
        map = new HashMap<>();  // 直接 new 一个 HashMap
    }

    public boolean add(E e) {
        return map.put(e, PRESENT) == null;  // 元素当 key 存进去
    }

    public boolean remove(Object o) {
        return map.remove(o) == PRESENT;
    }

    public boolean contains(Object o) {
        return map.containsKey(o);
    }

    public int size() {
        return map.size();
    }

    public boolean isEmpty() {
        return map.isEmpty();
    }
}
```

**本质：HashSet 就是 HashMap 的"马甲"。**

你把元素加到 HashSet，它实际上是把元素当成 **HashMap 的 key** 存进去，value 统一用一个固定的 `PRESENT` 对象占位。

---

### 核心原理

```java
// 创建 HashSet
Set<String> set = new HashSet<>();

// 执行 add("A")
set.add("A");
// → 内部调用：map.put("A", PRESENT)
//   key = "A"（你加的元素）
//   value = PRESENT（一个固定的 Object 对象）

// 执行 add("A") 第二次
set.add("A");
// → 内部调用：map.put("A", PRESENT)
//   HashMap 发现 key "A" 已经存在
//   → 覆盖 value（还是 PRESENT）
//   → 返回旧 value（不是 null）
//   → HashSet.add() 返回 false → 加不进去
```

```
内存结构：

HashSet                          HashMap
┌──────────┐                   ┌──────────────────┐
│ add("A") │ ──→ map.put("A") ──→ │ key     │ value  │
│ add("B") │ ──→ map.put("B") ──→ │ ────────┼──────── │
│ add("A") │ ──→ map.put("A") ──→ │ "A"     │ PRESENT│
└──────────┘     （第二次）        │ "B"     │ PRESENT│
                                   │ ...     │ ...    │
                                   └──────────────────┘
```

---

### 关键问题

#### 1. 为什么添加重复元素时返回 false？

```java
Set<String> set = new HashSet<>();
System.out.println(set.add("A"));  // true  （第一次，加进去了）
System.out.println(set.add("A"));  // false （第二次，没加进去）
```

原因看 HashMap.put() 的返回值：

```java
// HashMap.put() 返回：被替换的旧 value，如果没有旧值就返回 null
public V put(K key, V value) {
    // ... 如果 key 已存在，替换 value，返回旧 value
    // ... 如果 key 不存在，插入新节点，返回 null
}

// HashSet.add()：
public boolean add(E e) {
    return map.put(e, PRESENT) == null;
    // 第一次：key 不存在 → put 返回 null → null == null → true
    // 第二次：key 已存在 → put 返回旧 PRESENT → PRESENT == null → false
    // 但是！如果 PRESENT 不是 null，这行代码就有问题
}
```

等等，这里有个细节——`PRESENT` 是个 `new Object()`，它不是 null。那第二次 `map.put("A", PRESENT)` 返回的是第一次存的旧 `PRESENT`，不是 null。所以 `PRESENT == null` 结果是 `false`——正确！

但如果存进去的 value 正好就是 null 呢？不用担心，`new Object()` 永远不是 null。

#### 2. HashSet 怎么判断两个元素是否重复？

这就回到了 HashMap 判断 key 是否重复的方式：

```java
// 1. 先比 hashCode()
//    如果 hashCode 不同 → 肯定不重复
//    如果 hashCode 相同 → 进入第 2 步

// 2. 再比 equals()
//    如果 equals 也相同 → 同一个 key → 覆盖
//    如果 equals 不同 → 哈希冲突 → 挂链表/红黑树
```

```java
// 自定义对象存 HashSet 必须重写 hashCode 和 equals
class Person {
    String name;

    // 如果不重写 hashCode + equals：
    // new Person("张三") 和 new Person("张三") 会被当成两个不同的对象！
}
```

#### 3. HashSet 允许 null 吗？

允许，因为 HashMap 允许 key 为 null：

```java
Set<String> set = new HashSet<>();
set.add(null);   // ✅ 可以
set.add(null);   // 加不进去，返回 false
System.out.println(set.size());  // 1
```

#### 4. HashSet 是线程安全的吗？

不是，和 HashMap 一样线程不安全。

```java
// 多线程下需要包装
Set<String> set = Collections.synchronizedSet(new HashSet<>());
// 或者用并发包下的
Set<String> set = new CopyOnWriteArraySet<>();
```

#### 5. HashSet 的遍历顺序

```java
Set<String> set = new HashSet<>();
set.add("C");
set.add("A");
set.add("B");
System.out.println(set);  // 可能是 [A, B, C] 也可能是 [C, A, B]
```

底层是 HashMap，所以遍历顺序就是 HashMap 的 key 遍历顺序——**不保证有序**。而且扩容后顺序可能还会变。

### 面试回答

> **面试官：说一下 HashSet 的实现原理？**
>
> HashSet 底层是 HashMap。加元素时，把元素当 HashMap 的 key 存进去，value 统一用一个固定的 PRESENT 对象占位。
>
> 因为 HashMap 的 key 不能重复（靠 hashCode + equals 判断），所以 HashSet 就天然有了去重能力。
>
> 所以 HashSet 的特性都继承自 HashMap：不保证顺序、允许 null、线程不安全。

### 面试简答版

- **HashSet 底层就是 HashMap**，元素当 key，value 统一用 PRESENT 占位
- 去重原理：利用 HashMap key 不可重复的特性（hashCode + equals）
- `add("A")` → `map.put("A", PRESENT)`，重复时返回 false
- 允许 null、不保证顺序、线程不安全
- 自定义对象存 HashSet 要重写 `hashCode()` + `equals()`

## 2. HashSet 如何检查重复？如何保证数据不可重复？

### 先讲场景

第 1 题说了 HashSet 底层是 HashMap，去重靠的就是 HashMap 的 key 不能重复。

但重复是怎么判断的？是两个对象"内容一样"就算重复，还是"必须是同一个对象"才算重复？

```java
Person p1 = new Person("张三");
Person p2 = new Person("张三");

set.add(p1);
set.add(p2);
// 这两个"张三"算重复吗？
```

答案：**取决于你有没有正确重写 `hashCode()` 和 `equals()`**。

---

### HashSet 判断重复的两步流程

```text
add(e) 时，HashMap 判断 key 是否存在的流程：

第一步：计算 e.hashCode()
        ↓
       用 hashCode 定位到数组的某个桶
        ↓
       桶是空的？       → ✅ key 不存在，直接插入
       桶里有元素？      → 进入第二步

第二步：比较 e.equals(桶里已有元素)
         ↓
        equals 相等？   → ❌ key 已存在，覆盖 value，返回 false
        equals 不相等？ → ✅ 哈希冲突，挂链表/红黑树
```

**关键：先比 hashCode，再比 equals。**

- hashCode **不同** → 肯定不是同一个 key，直接进不同桶
- hashCode **相同** → 进入同一个桶，再用 equals 判断是否真的相等

---

### 如果不重写 hashCode 会怎样？

```java
class Person {
    String name;
    
    Person(String name) {
        this.name = name;
    }
    // ❌ 没有重写 hashCode 和 equals
}

Set<Person> set = new HashSet<>();
set.add(new Person("张三"));
set.add(new Person("张三"));
System.out.println(set.size());  // 2 ❌ 内容一样，但没去重！
```

**为什么？**

`Object` 默认的 `hashCode()` 返回的是**内存地址转换来的整数**，两个 `new Person("张三")` 是不同的对象，内存地址不同，hashCode 不同。

```
new Person("张三")  → hashCode = 12345  → 进桶 3
new Person("张三")  → hashCode = 67890  → 进桶 7

两个不同的桶，HashSet 认为它们"完全不相关"→ 都加进去了
```

---

### 如果只重写 equals 不重写 hashCode 呢？

```java
class Person {
    String name;
    
    Person(String name) {
        this.name = name;
    }
    
    @Override
    public boolean equals(Object obj) {  // ✅ 只重写了 equals
        if (this == obj) return true;
        if (obj instanceof Person p) {
            return this.name.equals(p.name);
        }
        return false;
    }
    // ❌ 没有重写 hashCode
}

Set<Person> set = new HashSet<>();
set.add(new Person("张三"));
set.add(new Person("张三"));
System.out.println(set.size());  // 还是 2 ❌
```

**为什么 equals 说相等了，还是没去重？**

因为 HashSet **先看 hashCode**！两个对象的 hashCode 不同（默认用内存地址算的），直接分到了不同的桶，**根本没机会走到 equals 那一步**。

```
对象 1：hashCode = 123 → 进桶 1
对象 2：hashCode = 456 → 进桶 5
                          ↓
               equals 根本不会被调用！
```

---

### ✅ 正确做法：一起重写

```java
class Person {
    String name;
    
    Person(String name) {
        this.name = name;
    }
    
    @Override
    public boolean equals(Object obj) {
        if (this == obj) return true;
        if (obj instanceof Person p) {
            return this.name.equals(p.name);
        }
        return false;
    }
    
    @Override
    public int hashCode() {
        return Objects.hash(name);  // 用 name 算 hashCode
    }
}

Set<Person> set = new HashSet<>();
set.add(new Person("张三"));
set.add(new Person("张三"));
System.out.println(set.size());  // 1 ✅ 正确去重
```

**执行过程：**

```
两个 Person("张三") 的 name 相同
→ hashCode 相同（都是 name.hashCode()）
→ 进了同一个桶
→ equals 比较发现内容相同
→ 认为是同一个 key → 去重 ✅
```

---

### 补充：哈希冲突的情况

```java
// hashCode 相同，但 equals 不相等的情况
// 比如两个不同的字符串恰好 hashCode 一样
"Aa".hashCode();   // 2112
"BB".hashCode();   // 2112

// 虽然 hashCode 一样（进了同一个桶）
// 但 equals 判断为不同元素
// → 挂链表，两个都能存进去
```

这种叫 **哈希冲突**，HashMap 会用链表/红黑树处理，不影响去重的正确性。

---

### 总结

| 情况 | hashCode | equals | HashSet 能去重吗？ |
|:---|:---:|:---:|:---:|
| 都不重写 | 内存地址（不同） | 内存地址（不同） | ❌ 不能 |
| 只重写 equals | 内存地址（不同） | 内容相同 | ❌ **不能！**（先比 hashCode 就淘汰了） |
| 两个都重写 | 内容相同 | 内容相同 | ✅ **正确去重** |

### 面试回答

> **面试官：HashSet 如何检查重复？**
>
> HashSet 底层是 HashMap，检查重复靠两步：先算 hashCode，再比 equals。
>
> hashCode 决定元素进哪个桶，equals 决定同一个桶里是否真的相等。只有 hashCode 和 equals 都相等，才认为是重复元素。
>
> 所以自定义对象存 HashSet 必须同时重写 hashCode 和 equals。如果只重写 equals，hashCode 还是用内存地址算的，两个内容相同的对象会被分到不同桶，根本不会触发 equals 比较。

### 面试简答版

- 两步检查：**先 hashCode → 再 equals**
- hashCode 不同 → 直接进不同桶，不认为重复
- hashCode 相同 + equals 相同 → 重复
- hashCode 相同 + equals 不同 → 哈希冲突，挂链表，不认为是重复
- **自定义对象必须同时重写 hashCode 和 equals**，缺一个都可能导致去不了重

## 3. hashCode() 与 equals() 的相关规定

### 先讲场景

第 2 题讲了"必须同时重写 hashCode 和 equals"，但没有系统地说清楚它俩之间的"契约"到底是什么。

这题就是专门整理这组规定的。

---

### 五条规定（记住这五条就够了）

**规定 1：如果两个对象 equals 相等，hashCode 必须相等**

```java
Person a = new Person("张三");
Person b = new Person("张三");

a.equals(b) == true   // 如果相等
→ a.hashCode() == b.hashCode()  // 必须相等
```

**规定 2：如果两个对象 equals 相等，equals 方法返回 true**

这是 equals 的定义，不说自明。

**规定 3：如果两个对象 hashCode 相等，它们不一定 equals 相等**

```java
// hashCode 相等 ≠ 同一个对象

"Aa".hashCode();   // 2112
"BB".hashCode();   // 2112

"Aa".equals("BB"); // false ← hashCode 一样，但不是同一个内容
```

这就叫**哈希冲突**，正常现象，HashMap 用链表/红黑树处理。

**规定 4：equals 被重写过，hashCode 也必须重写**

这是规定 1 的推论。不遵守的话：

```java
// 只重写 equals，不重写 hashCode
// → a.equals(b) 返回 true
// → a.hashCode() != b.hashCode()（还是内存地址算的）
// → 违反了规定 1，HashSet/HashMap 无法正常工作
```

**规定 5：不重写 hashCode 的话，两个对象永远不会相等**

```java
// hashCode 的默认实现是根据内存地址算的
// 就算两个对象内容完全相同，只要不是同一个对象，hashCode 就不同

Person a = new Person("张三");
Person b = new Person("张三");

a.hashCode();  // 12345  （内存地址换算的）
b.hashCode();  // 67890  （不同对象，不同地址）
```

所以如果不重写 hashCode，new 出来的两个对象 hashCode 永远不同，HashSet 永远认为它们不是重复元素。

---

### 补充：== 和 equals 的区别

这也是面试必连问的：

```java
String a = new String("Hello");
String b = new String("Hello");
String c = a;

// == ：比较内存地址（是不是同一个对象）
System.out.println(a == b);   // false ← 两个不同对象，地址不同
System.out.println(a == c);   // true  ← c 指向 a，是同一个地址

// equals：比较内容（值是不是一样）
System.out.println(a.equals(b));  // true ← 内容都是 "Hello"
```

| 对比 | == | equals |
|:---|:---|:---|
| **比较什么** | 内存地址 | 内容值（默认也是地址，但 String 等类重写了） |
| **两个 new String("A")** | false（不同地址） | true（内容相同） |
| **同一个对象** | true | true |
| **能自定义吗** | ❌ 不能 | ✅ 可以重写 |

**重点：** `==` 比较的是**引用指向的内存地址**，`equals` 比较的是**对象的内容**。String 默认重写了 equals，所以 `"Hello".equals("Hello")` 是 true。

---

### 完整规则总结

```
hashCode 相等  ≠  equals 相等（可能有哈希冲突）
equals 相等    →  hashCode 必须相等（硬性规定）

重写 equals    →  必须重写 hashCode
不重写 hashCode →  两个对象永远不相等（对 HashSet/HashMap 来说）

== 比地址  equals 比内容
```

### 面试简答版

- equals 相等 → hashCode 必须相等（硬规定）
- hashCode 相等 ≠ equals 相等（哈希冲突）
- 重写 equals 必须重写 hashCode
- 不重写 hashCode，HashSet/HashMap 无法正确去重
- `==` 比地址，`equals` 比内容

## 4. HashSet 与 HashMap 的区别

### 先讲场景

看了第 1 题你就知道——HashSet 底层就是 HashMap。

那问题来了：**既然底层一样，为什么还要分两个东西？它们到底有什么区别？**

答案很简单：**用途不同。**

| | 存东西的姿势 | 你要解决什么问题 |
|:---|:---|:---|
| **HashMap** | 存 key-value 对 | "根据 key 找 value"（字典查询） |
| **HashSet** | 只存元素（value 是固定的） | "帮我自动去重" |

---

### 对比表

| 对比维度         | HashMap                     | HashSet                     |     |
| :----------- | :-------------------------- | :-------------------------- | --- |
| **实现的接口**    | Map                         | Set                         |     |
| **存储内容**     | **键值对**（key-value）          | **单个元素**（其实是 HashMap 的 key） |     |
| **重复规则**     | key 不能重复，value 可以重复         | 元素不能重复                      |     |
| **能否 get()** | ✅ `map.get(key)` 查 value    | ❌ 没有 get()，只能遍历或 contains   |     |
| **底层**       | 数组 + 链表 + 红黑树               | **就是 HashMap**（借来用的）        |     |
| **存元素的方法**   | `put(key, value)`           | `add(e)`                    |     |
| **性能**       | 基本一样（因为 HashSet 就是 HashMap） |                             |     |

---

### 核心区别：存法不同

```java
// HashMap：存键值对
Map<String, Integer> map = new HashMap<>();
map.put("张三", 90);     // key = 张三, value = 90
map.put("李四", 85);     // key = 李四, value = 85
map.get("张三");         // 90

// HashSet：只存单个元素
Set<String> set = new HashSet<>();
set.add("张三");          // 只存元素本身
set.add("李四");
// set.get("张三")        // ❌ 不能 get
set.contains("张三");     // ✅ 只能用 contains 判断有没有
```

**HashMap 是"查字典"：你给 key，我告诉你 value。**
**HashSet 是"签到表"：你只关心"这个人有没有来"，不关心别的信息。**

---

### 本质：HashSet 就是 HashMap 套了一层

```java
// HashSet 内部
public class HashSet<E> {
    private HashMap<E, Object> map;
    private static final Object PRESENT = new Object();
    
    public boolean add(E e) {
        return map.put(e, PRESENT) == null;
    }
}
```

画成图是这样的：

```
HashSet.add("A")
    ↓
HashMap.put("A", PRESENT)
    ↓
                      ┌──────────────────────┐
                      │   HashMap              │
                      │  key    │  value       │
                      │ ────────┼────────────  │
                      │  "A"    │  PRESENT     │
                      │  "B"    │  PRESENT     │
                      │  "C"    │  PRESENT     │
                      └──────────────────────┘
                      
你关心的：         A 存进去了    ✅
HashSet 不关心的： value 是啥？  无所谓，反正固定
```

### 什么时候用哪个？

```java
// 需要根据 key 查 value → HashMap
// 场景：用户名查密码、学号查成绩、商品ID查商品详情
Map<String, User> userMap = new HashMap<>();
userMap.put("user001", user);
User u = userMap.get("user001");  // ✅ 查得到

// 只需要判断"有没有"、需要自动去重 → HashSet
// 场景：已访问的 URL 集合、黑名单、在线用户ID集合
Set<String> visitedUrls = new HashSet<>();
visitedUrls.add("/home");
visitedUrls.add("/about");
visitedUrls.contains("/home");     // true ✅ 只能判断有没有
```

### 一句话总结

> **HashMap 存键值对，为了"查"；HashSet 存单个元素，为了"去重"。**
>
> 但 HashSet 底层就是 HashMap——把元素当 key，value 全用同一个固定对象占位。

### 面试简答版

- HashMap → Map 接口，存 key-value，用 `put`/`get`
- HashSet → Set 接口，存单个元素，用 `add`/`contains`
- HashSet 底层就是 HashMap（元素当 key，value 用 PRESENT 占位）
- HashMap 为了"查"，HashSet 为了"去重"
