# Collection 接口

## 1. 迭代器 Iterator 是什么？

### 先讲场景

假设你有一个 `ArrayList`，想遍历它：

```java
List<String> list = new ArrayList<>(List.of("A", "B", "C"));
for (int i = 0; i < list.size(); i++) {
    System.out.println(list.get(i));
}
```

这没问题，因为 List 有索引，能 `get(i)`。

但如果你换成 `HashSet` 呢？

```java
Set<String> set = new HashSet<>(Set.of("A", "B", "C"));
for (int i = 0; i < set.size(); i++) {
    System.out.println(set.get(i));  // ❌ Set 没有 get() 方法！
}
```

Set 没有索引，没法用普通 for 循环。那怎么遍历所有集合？

**Iterator 就是干这个的**——不管 List、Set、Queue，它提供一套统一的遍历方式。

---

### 核心理解

Iterator 就是一个"游标"，你可以想象成**手里的遥控器**，对着集合按：

- `hasNext()` → 问"还有没有下一个？"（有返回 true，没有返回 false）
- `next()` → "把下一个给我"（返回当前元素，游标往前走一步）
- `remove()` → "把刚才给的那个删掉"

```java
Iterator<String> it = set.iterator();
while (it.hasNext()) {        // 还有下一个吗？
    String s = it.next();      // 给我下一个
    System.out.println(s);
}
```

**输出（顺序不固定，因为 HashSet 无序）：**

```
A
C
B
```

每次调 `next()`，游标就往前走一步：

```
初始状态：游标 ▎A  B  C
next() 后：A ▎B  C      ← 返回 A
next() 后：A  B ▎C       ← 返回 B
next() 后：A  B  C ▎     ← 返回 C
next() 后：抛异常（没有下一个了）
```

---

### 增强 for 循环就是 Iterator 的"语法糖"

你平时写的：

```java
for (String s : set) {
    System.out.println(s);
}
```

编译后本质就是：

```java
for (Iterator<String> it = set.iterator(); it.hasNext(); ) {
    String s = it.next();
    System.out.println(s);
}
```

所以增强 for 循环能遍历任何 `Collection`（List、Set、Queue 都行），就是因为它们都有 `iterator()` 方法。

---

### 那遍历时想删元素怎么办？

普通 for 循环用 `list.remove()` 会抛 `ConcurrentModificationException`（回忆第 9 节的 fail-fast）。用 Iterator 自己的 `remove()` 就没问题：

```java
Iterator<String> it = list.iterator();
while (it.hasNext()) {
    String s = it.next();
    if ("B".equals(s)) {
        it.remove();  // ✅ 删当前元素，不会抛异常
    }
}
```

原理：`it.remove()` 会同步更新 `expectedModCount`，所以迭代器下次 `next()` 时不会发现异常。

---

### 面试常问点

**1. Iterator 和 ListIterator 的区别？**

`ListIterator` 是 Iterator 的"升级版"，只适用于 List：

```java
ListIterator<String> it = list.listIterator();
it.hasPrevious();    // 能往前遍历 ← Iterator 做不到
it.previous();       // 获取上一个
it.add("X");         // 能添加元素   ← Iterator 做不到
it.set("Y");         // 能修改元素   ← Iterator 做不到
```

| 对比 | Iterator | ListIterator |
|------|----------|-------------|
| 适用范围 | 所有 Collection | 只有 List |
| 遍历方向 | 只能向后 | 可以双向 |
| 能不能 add | ❌ | ✅ |
| 能不能 set | ❌ | ✅ |

**2. Iterator 遍历过程中集合被改了会怎样？**

会抛 `ConcurrentModificationException`（fail-fast 机制，[[03集合容器概述#9. Java 集合的快速失败机制 "fail-fast"|第 9 题]]讲过）。

```java
Iterator<String> it = list.iterator();
it.next();
list.add("X");    // 遍历时外部改了集合
it.next();        // ❌ ConcurrentModificationException
```

---

### 面试回答

> **面试官：迭代器 Iterator 是什么？**
>
> Iterator 是 Java 集合的"统一遍历工具"。因为不同集合底层结构不一样（List 有索引、Set 没有），但 Iterator 让你用同样的方式遍历所有集合。
>
> 核心就三个方法：`hasNext()` 判断有没有下一个，`next()` 取出下一个，`remove()` 删除当前元素。
>
> 增强 for 循环底层就是 Iterator 的语法糖。另外注意，遍历时如果要删元素，必须用迭代器自己的 `remove()`，直接调集合的 `remove()` 会抛异常。

### 面试简答版

- Iterator 是集合的统一遍历方式，任何 Collection 都能用
- 三个方法：`hasNext()`、`next()`、`remove()`
- 增强 for 循环底层就是 Iterator
- 遍历时删元素要用 `it.remove()`，不要用 `list.remove()`
- `ListIterator` 是 Iterator 的升级版，支持双向遍历 + add + set

---

## 2. Iterator 怎么使用？有什么特点？

### 先讲场景

[[#1. 迭代器 Iterator 是什么？|第 1 题]]讲了 Iterator"是什么"，但实际写代码时，你会发现有**好几种写法**都能遍历集合，而且 Java 8 之后又多了一种新写法。

这一题专门讲"怎么用"和"有什么特点"。

---

### 使用方式一：标准 Iterator 循环（最原始）

```java
Iterator<String> it = list.iterator();
while (it.hasNext()) {
    String s = it.next();
    System.out.println(s);
}
```

**特点**：
- 你手动控制 Iterator 的创建、判断、取值
- 遍历过程中可以 `remove()`
- 这种写法叫**外部迭代**——你在外面写循环，一步步"拉"数据

---

### 使用方式二：增强 for 循环（最常用）

```java
for (String s : list) {
    System.out.println(s);
}
```

**特点**：
- 省事，不用自己写 `hasNext()` 和 `next()`
- 底层还是 Iterator（编译后自动转成方式一）
- 但遍历时不能 `remove()`（你拿不到 Iterator 的引用）

---

### 使用方式三：forEachRemaining()（JDK 8，一步到位）

```java
list.iterator().forEachRemaining(s -> System.out.println(s));
```

**特点**：
- 拿到 Iterator 后，直接告诉它"把剩下的元素全部按这个规则处理"
- 适合对集合做**同一操作**（比如全部打印、全部转大写）
- 内部自动循环，你不需要写 `while` 了

```java
// 举个例子：把集合里所有元素转大写
List<String> list = new ArrayList<>(List.of("a", "b", "c"));
list.iterator().forEachRemaining(s -> System.out.println(s.toUpperCase()));
// 输出：A B C
```

---

### 使用方式四：Collection.forEach() （JDK 8，最现代）

```java
list.forEach(s -> System.out.println(s));
// 或者方法引用
list.forEach(System.out::println);
```

**特点**：
- 不是用 Iterator，是 Collection 接口自带的方法
- 这种叫**内部迭代**——你把"干什么"告诉集合，循环由集合内部自己控制
- 代码最短，现代 Java 最推荐

---

### 外部迭代 vs 内部迭代

| 对比 | 外部迭代（Iterator / 增强 for） | 内部迭代（forEach） |
|------|:---:|:---:|
| **谁控制循环** | 你控制 | 集合内部控制 |
| **遍历时能不能删** | ✅ Iterator 可以 | ❌ 不能 |
| **能不能 break/return** | ✅ 可以 | ❌ 不行（除非抛异常） |
| **代码简洁度** | 一般 | 简洁 |
| **适合场景** | 需要中途退出、需要删元素 | 全部元素做同一操作 |

```java
// 外部迭代：你想在遍历到 "B" 时停下来
for (String s : list) {
    if ("B".equals(s)) break;  // ✅ 可以
    System.out.println(s);
}

// 内部迭代：没法中途停下来
list.forEach(s -> {
    if ("B".equals(s)) return;  // ❌ 这只是跳过本次，不是 break
    System.out.println(s);
});
```

---

### Iterator 的核心特点总结

| 特点 | 解释 |
|------|------|
| **单向遍历** | 只能从前往后，不能回头（想回头用 ListIterator） |
| **统一接口** | 不管底层是数组还是链表，用 Iterator 都是一样的写法 |
| **fail-fast** | 遍历时集合结构被改了，立即抛异常（[[03集合容器概述#9. Java 集合的快速失败机制 "fail-fast"|第 9 题]]） |
| **可删除** | 唯一能在遍历时安全删元素的方式（`it.remove()`） |
| **只读访问** | 只能读和删，不能改已有元素，不能新增（ListIterator 除外） |
| **延迟获取** | `next()` 才取一个，不是一次性全加载到内存 |

### 面试回答

> **面试官：Iterator 怎么使用？有什么特点？**
>
> 使用方式有几种：
>
> 最原始的是 `while(it.hasNext())` + `it.next()` 手动控制；最常用的是增强 for 循环，底层其实就是 Iterator 的语法糖；JDK 8 之后还可以用 `forEachRemaining()` 和集合的 `forEach()` 方法。
>
> Iterator 是**外部迭代**——你在外面写循环一个个拉数据。而 `forEach()` 是**内部迭代**——你把处理逻辑传进去，集合自己控制循环。
>
> 核心特点：单向遍历、统一接口、fail-fast、遍历时可删除元素。

### 面试简答版

- 四种用法：`while` 循环 / 增强 for / `forEachRemaining()` / `forEach()`
- 外部迭代（Iterator）：你控制循环，能 break、能 remove
- 内部迭代（forEach）：集合自己循环，代码简洁，但不能 break/remove
- 特点：单向、统一、fail-fast、可删除、只读

---

## 3. 如何边遍历边移除 Collection 中的元素？

### 先讲场景

你有一个名单，想遍历一遍，把叫"张三"的踢出去：

```java
List<String> list = new ArrayList<>(List.of("小明", "张三", "小红", "张三"));
```

如果直接写，你很可能踩坑。

---

### ❌ 错误写法 1：增强 for 循环直接 remove

```java
for (String s : list) {
    if ("张三".equals(s)) {
        list.remove(s);  // ❌ ConcurrentModificationException
    }
}
```

**为什么报错？** 之前 [[03集合容器概述#9. Java 集合的快速失败机制 "fail-fast"|第 9 题]]讲过了：增强 for 底层是 Iterator，遍历时集合的 `modCount` 变了，Iterator 检查发现不一致，直接抛异常。

---

### ❌ 错误写法 2：普通 for 循环正着删

```java
List<String> list = new ArrayList<>(List.of("小明", "张三", "张三", "小红"));
for (int i = 0; i < list.size(); i++) {
    if ("张三".equals(list.get(i))) {
        list.remove(i);
    }
}
System.out.println(list); // [小明, 张三, 小红] ← 还有一个张三没删掉！
```

**为什么不报错但结果不对？**

看执行过程：

```
初始： [小明, 张三, 张三, 小红]    i=0 → i=1 → i=2 → i=3

i=0：小明   → 不删 → i 变成 1
i=1：张三① → 删掉 → [小明, 张三②, 小红]  ← 张三②前移到了 i=1
                      但 i 继续走到 i=2，跳过了 i=1！
i=2：小红   → 不删
```

**关键问题**：删除了 i=1 的元素后，后面的元素往前移，原来在 i=2 的张三②移到了 i=1，但 i 已经变成 2 了，**张三②被跳过了**。

这就是"正着删会漏元素"的问题。只有连续相邻的两个目标元素时才会触发，但面试就爱考这个坑。

---

### ✅ 正确写法 1：用 Iterator.remove()

```java
Iterator<String> it = list.iterator();
while (it.hasNext()) {
    String s = it.next();
    if ("张三".equals(s)) {
        it.remove();  // ✅ 安全删除
    }
}
System.out.println(list); // [小明, 小红]
```

**为什么可以？** `it.remove()` 会把 `expectedModCount` 同步更新，下次 `next()` 时两个值一致，不会抛异常。

---

### ✅ 正确写法 2：用 Collection.removeIf()（JDK 8，最推荐）

```java
list.removeIf(s -> "张三".equals(s));
```

一行搞定。**实际开发首选这种写法。**

原理和 Iterator 一样，也是遍历时删除，但 JDK 帮你封装好了。

---

### ✅ 正确写法 3：普通 for 循环倒着删

```java
for (int i = list.size() - 1; i >= 0; i--) {
    if ("张三".equals(list.get(i))) {
        list.remove(i);
    }
}
```

倒着删不会影响还没遍历到的位置，因为删的是已经检查过的。但注意：**这种方法只适用于 List**（有索引），Set 不能用。

---

### 四种方式对比

| 方式 | 代码量 | 推荐程度 | 适用范围 |
|------|:---:|:---:|---------|
| `removeIf()` | 1 行 | ⭐⭐⭐ 最推荐 | 任何 Collection |
| `Iterator.remove()` | 5 行 | ⭐⭐ 备用 | 任何 Collection |
| for 循环倒着删 | 4 行 | ⭐ 仅限 List | 只有 List |
| 增强 for 直接删 | — | ❌ 不能用 | — |

### 面试回答

> **面试官：如何边遍历边移除 Collection 中的元素？**
>
> 有三种正确方式：
>
> 最推荐的是 JDK 8 的 `removeIf()`，一行代码搞定，简洁安全，任何 Collection 都能用。
>
> 其次是手动用 Iterator 的 `remove()` 方法，原理是它会同步更新 `expectedModCount`，不会触发 fail-fast。
>
> 如果是 List，也可以用普通 for 循环倒着删，不会漏元素。
>
> 注意两点易错：增强 for 循环直接调 `list.remove()` 会抛异常；普通 for 正着删会漏掉连续相邻的元素。

### 面试简答版

- 最推荐：`list.removeIf(条件)` — 一行搞定
- 其次：`Iterator.remove()` — 遍历时安全删除
- 备选（仅 List）：for 循环倒着删
- 错误：增强 for 直接 `list.remove()` 会抛异常；普通 for 正着删会漏删

## 4. Iterator 和 ListIterator 有什么区别？

### 先讲场景

第 1 题的面试常问点里已经简单提过两者的区别，但你现在可能会想：

> "Iterator 只能往后走，如果我遍历到一半想**回头**怎么办？"
> "Iterator 只能删元素，如果我遍历时想**改值**或者**加元素**呢？"

这就是 `ListIterator` 存在的意义——它是 Iterator 的"完全体"，但**只给 List 用**。

```java
// Iterator：能做的事
it.hasNext()    // 还有没有下一个？
it.next()       // 取下一个
it.remove()     // 删当前（可选）

// ListIterator：上面都能做 + 还能做下面这些
it.hasPrevious()    // 还有没有上一个？
it.previous()       // 取上一个
it.nextIndex()      // 下一个元素的索引
it.previousIndex()  // 上一个元素的索引
it.add(x)           // 加元素
it.set(x)           // 改当前元素
```

---

### 核心区别

#### 区别 1：适用范围不同

```java
Iterator<String> it = list.iterator();     // ✅ List 能用
Iterator<String> it = set.iterator();      // ✅ Set 也能用
Iterator<String> it = queue.iterator();    // ✅ Queue 也能用

ListIterator<String> lit = list.listIterator();  // ✅ 只有 List 能用
ListIterator<String> lit = set.listIterator();   // ❌ Set 没有这个方法
```

`Iterator` 是所有 `Collection` 都能用的"通用工具"。`ListIterator` 是"豪华版"，只有 List 有。

#### 区别 2：遍历方向（最直观的区别）

```java
List<String> list = List.of("A", "B", "C");

// Iterator：只能从头到尾
Iterator<String> it = list.iterator();
it.next();  // A
it.next();  // B
// 想回头拿 A？做不到

// ListIterator：可以从尾到头
ListIterator<String> lit = list.listIterator();
lit.next();      // A
lit.next();      // B
lit.previous();  // ← 回到 B！     Iterator 做不到
lit.previous();  // ← 回到 A！
```

想象 Iterator 是**单向自动扶梯**，只能往前走。ListIterator 是**升降电梯**，想上想下都行。

#### 区别 3：能不能修改元素

```java
// Iterator：只能读和删，不能改
it.next();
it.remove();   // ✅ 能删
it.set("X");   // ❌ Iterator 没有 set()

// ListIterator：能读、能删、能改、能加
lit.next();
lit.set("X");  // ✅ 把当前元素替换成 "X"
lit.add("Y");  // ✅ 在当前游标位置插入 "Y"
```

#### 区别 4：可以从任意位置开始

```java
// Iterator：只能从头开始
Iterator<String> it = list.iterator();  // 游标在位置 0 之前

// ListIterator：可以从指定索引开始
ListIterator<String> lit = list.listIterator(2);  // 游标从索引 2 开始
lit.next();  // 直接拿到索引 2 的元素（第三个元素）

// 甚至可以从尾部往前遍历
ListIterator<String> lit2 = list.listIterator(list.size());  // 游标在末尾
while (lit2.hasPrevious()) {
    System.out.println(lit2.previous());  // C → B → A（逆序遍历）
}
```

---

### 完整对比表

| 对比 | Iterator | ListIterator |
|------|:--------:|:------------:|
| 适用范围 | 所有 Collection | **只有 List** |
| 遍历方向 | 只能往后（单向） | **能前能后（双向）** |
| `remove()` | ✅ | ✅ |
| `set()` | ❌ | ✅ **可以改元素** |
| `add()` | ❌ | ✅ **可以加元素** |
| 获取前后索引 | ❌ | ✅ `nextIndex()` / `previousIndex()` |
| 指定起始位置 | ❌ 只能从头开始 | ✅ `listIterator(索引)` |

### 用 ListIterator 逆序遍历的实用例子

```java
List<String> list = new ArrayList<>(List.of("A", "B", "C", "D"));

// 逆序输出
ListIterator<String> lit = list.listIterator(list.size());
while (lit.hasPrevious()) {
    System.out.print(lit.previous() + " ");  // D C B A
}
```

### 面试回答

> **面试官：Iterator 和 ListIterator 有什么区别？**
>
> 主要有四个区别：
>
> 第一，**适用范围**：Iterator 所有 Collection 都能用，ListIterator 只能 List 用。
>
> 第二，**遍历方向**：Iterator 只能往后走，ListIterator 可以前可以后。
>
> 第三，**修改能力**：Iterator 只能删元素（remove），ListIterator 还能改（set）和加（add）。
>
> 第四，**起始位置**：Iterator 必须从头开始，ListIterator 可以指定起始索引。
>
> 简单记：ListIterator 是 Iterator 的升级版，功能更强，但只给 List 用。

### 面试简答版

- Iterator 通用（所有 Collection），ListIterator 专用（只有 List）
- Iterator 单向，ListIterator 双向
- Iterator 只能删，ListIterator 能删能改能加
- Iterator 从头开始，ListIterator 可以从任意位置开始

---

## 5. 遍历一个 List 有哪些不同的方式？每种方法的实现原理是什么？Java 中 List遍历的最佳实践是什么？

### 先讲场景

你已经知道了 Iterator 和增强 for 能遍历集合，但 List 比较特殊——它有索引。

所以遍历 List 的方式比 Set 更多，而且**不同的方式性能差距很大**，面试也爱问。

---

### 方式一：普通 for 循环（按索引）

```java
List<String> list = new ArrayList<>(List.of("A", "B", "C"));
for (int i = 0; i < list.size(); i++) {
    System.out.println(list.get(i));
}
```

**原理**：直接通过索引访问底层数组，`list.get(i)` 一步到位。

**⚠️ 注意**：这只适合 **ArrayList**，不适合 **LinkedList**。因为 `LinkedList.get(i)` 每次都要从头遍历到第 i 个节点，时间复杂度 O(n)，套在 for 循环里就是 O(n²)，巨慢。

```java
// 千万别这么写！
LinkedList<String> list = new LinkedList<>(List.of("A", "B", "C"));
for (int i = 0; i < list.size(); i++) {
    System.out.println(list.get(i));  // get(0) → 找头节点
                                      // get(1) → 从头走到第 1 个
                                      // get(2) → 从头走到第 2 个
                                      // ...每次都是 O(n)，总共 O(n²)
}
```

---

### 方式二：增强 for 循环

```java
for (String s : list) {
    System.out.println(s);
}
```

**原理**：编译后转成 Iterator。不管是 ArrayList 还是 LinkedList，都是**一次遍历走到底**，时间复杂度 O(n)。

---

### 方式三：显式 Iterator

```java
Iterator<String> it = list.iterator();
while (it.hasNext()) {
    String s = it.next();
    System.out.println(s);
}
```

**原理**：和增强 for 一样，但你能拿到 Iterator 的引用，遍历时可以 `remove()`。

---

### 方式四：ListIterator

```java
ListIterator<String> lit = list.listIterator();
while (lit.hasNext()) {
    System.out.println(lit.next());
}
// 还能逆序遍历
while (lit.hasPrevious()) {
    System.out.println(lit.previous());
}
```

**原理**：List 独有的迭代器，支持双向，还能 `set()` 和 `add()`。

---

### 方式五：forEach() + lambda

```java
list.forEach(s -> System.out.println(s));
// 或
list.forEach(System.out::println);
```

**原理**：**内部迭代**——集合内部自己遍历，你把"干什么"传进去就行。底层仍然是 for 循环遍历每个元素，但代码最简洁。

---

### 方式六：Stream API

```java
list.stream().forEach(s -> System.out.println(s));
list.stream().filter(s -> s.startsWith("A")).forEach(System.out::println);
```

**原理**：基于 `Spliterator`（可分割迭代器），支持**流水线操作**（过滤、映射、排序等）。不光是遍历，还能链式调用做复杂操作。

---

### 各种方式的对比

| 方式 | ArrayList 性能 | LinkedList 性能 | 能否 remove | 能否 break | 适用场景 |
|:---|:---:|:---:|:---:|:---:|:---|
| **普通 fori** | ✅ 最快（O(n)） | ❌ 巨慢（O(n²)） | ✅ | ✅ | 仅 ArrayList，需要索引时 |
| **增强 for** | ✅ O(n) | ✅ O(n) | ❌ | ✅ | 日常遍历，最常用 |
| **Iterator** | ✅ O(n) | ✅ O(n) | ✅ | ✅ | 需要遍历时删元素 |
| **ListIterator** | ✅ O(n) | ✅ O(n) | ✅+set/add | ✅ | 需要双向遍历 |
| **forEach()** | ✅ O(n) | ✅ O(n) | ❌ | ❌ | 遍历全部元素做同一操作 |
| **Stream** | ✅ O(n) | ✅ O(n) | ❌ | ❌ | 需要链式操作（过滤、映射） |

---

### List 遍历的最佳实践

核心原则：**看 List 底层是什么结构，再选遍历方式。**

Java 提供了一个 `RandomAccess` 接口来帮你判断：

```java
// RandomAccess 是一个"标记接口"——里面啥也没有
public interface RandomAccess {}

// ArrayList 实现了它
public class ArrayList<E> extends AbstractList<E>
        implements List<E>, RandomAccess, ...  // ✅ 标记了

// LinkedList 没有实现它
public class LinkedList<E> extends AbstractSequentialList<E>
        implements List<E>, ...                // ❌ 没标记
```

有 `RandomAccess` 标记 → 底层是数组 → 用 fori 最快
没有 `RandomAccess` 标记 → 底层是链表 → 用 Iterator / 增强 for

```java
// 通用写法：不管 ArrayList 还是 LinkedList 都安全
if (list instanceof RandomAccess) {
    for (int i = 0; i < list.size(); i++) {
        System.out.println(list.get(i));
    }
} else {
    for (String s : list) {
        System.out.println(s);
    }
}
```

实际开发中选遍历方式的速查：

```
├── 普通遍历（不需要删元素）
│   ├── 代码最简 → forEach() 或 增强 for
│   ├── ArrayList（有 RandomAccess）→ 普通 fori 最快
│   └── LinkedList（无 RandomAccess）→ 千万别用 fori！
│
├── 需要删元素
│   ├── 一行搞定 → removeIf()
│   └── 复杂条件 → Iterator.remove()
│
├── 需要双向遍历 → ListIterator
│
└── 需要过滤/转换 → Stream API
```

### 面试高频题：为什么普通 for 循环适合 ArrayList 但不适合 LinkedList？

```java
// ArrayList.get(i)：直接算内存地址取，O(1)
// 底层数组： [A, B, C, D, E]
//           get(3) → 直接拿第 4 个位置

// LinkedList.get(i)：从头一个个找，O(n)
// 底层链表： A → B → C → D → E
//           get(3) → 从 A 开始找，经过 B、C 才到 D
```

普通 for 循环 + ArrayList：**O(n)**（每次 get 是 O(1)，循环 n 次）
普通 for 循环 + LinkedList：**O(n²)**（每次 get 是 O(n)，循环 n 次）

```java
// 同样是 for 循环遍历 10 万个元素
ArrayList:  约 5ms   ✅
LinkedList: 约 5000ms ❌ 慢 1000 倍
```

所以面试官如果问你"遍历 LinkedList 用什么方式"，答案是 **增强 for 或 Iterator**，千万不要用 fori。

### 面试回答

> **面试官：遍历 List 有哪些方式？原理和最佳实践是什么？**
>
> 主要有 6 种方式：普通 fori、增强 for、Iterator、ListIterator、forEach()、Stream。
>
> 如果是 ArrayList，普通 fori 最快（底层是数组，索引直接访问）。但如果是 LinkedList，不能用 fori，因为每次 get(i) 都要从头遍历到 i，性能是 O(n²)，要改用增强 for 或 Iterator。
>
> 最佳实践：不需要删元素就用增强 for 或 forEach()；需要删元素用 removeIf() 或 Iterator；需要复杂操作（过滤、映射）用 Stream。

### 面试简答版

- 6 种方式：fori / 增强 for / Iterator / ListIterator / forEach / Stream
- ArrayList 用 fori 最快，LinkedList 千万不能用 fori（O(n²)）
- 不需要删 → 增强 for 或 forEach
- 需要删 → removeIf() 或 Iterator
- 需要复杂操作 → Stream

---

## 6. ArrayList 的优缺点

### 先讲场景

选 List 实现类时，90% 的情况你默认选 `ArrayList`。

但面试官会问你："为什么不总是用 ArrayList？什么时候该换别的？"

答案就是搞清楚它的优缺点。

---

### 优点

| 优点 | 为什么 |
|------|--------|
| **按索引查询快（O(1)）** | 底层是数组，直接算内存地址取，`get(i)` 一步到位 |
| **尾部插入快（O(1)）** | 直接在数组末尾放一个元素，不需要移动别的 |
| **内存连续、CPU 缓存友好** | 数组在内存里是一块连续空间，遍历时 CPU 预读效率高 |
| **省内存** | 相比 LinkedList，不用存前后指针，只存数据本身 |

```java
ArrayList 底层内存布局（示意）：
[ A | B | C | D | E | 空 | 空 | 空 | 空 | 空 ]
  0   1   2   3   4   5    6    7    8    9
```

`get(3)` → 直接算：数组首地址 + 3 × 元素大小 → 一步拿到 D

---

### 缺点

| 缺点 | 为什么 |
|------|--------|
| **中间插入/删除慢（O(n)）** | 插入/删除后，后面的元素要整体移位 |
| **扩容有开销** | 容量不够时新建数组 + 拷贝所有元素（1.5 倍扩容） |
| **不是线程安全的** | 多线程并发读写需要自己加锁或用 `CopyOnWriteArrayList` |
| **不能存基本类型** | 只能存对象，`int` 要装箱成 `Integer`（有性能损耗） |

#### 中间插入的过程

```java
List<String> list = new ArrayList<>(List.of("A", "B", "D", "E"));
// 想在索引 2 的位置插入 "C"：
list.add(2, "C");
```

```
插入前：[ A | B | D | E | 空 ]
                   ↑ 要在索引 2 插入

第一步：从索引 2 开始，后面的元素全部后移一位
       [ A | B | D | D | E ]   ← D 和 E 后移

第二步：在索引 2 放入 "C"
       [ A | B | C | D | E ]   ✅
```

插一个元素，后面的 n-2 个元素都要动。**平均要移动一半的元素**（n/2）。

#### 扩容的过程

```java
ArrayList 默认容量 10，存满 10 个后加第 11 个 → 扩容
```

```
扩容前： [ 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 ]  容量 10
                                                      满了
扩容：   新建一个容量 15 的数组（1.5 倍）
         把 10 个元素全部拷过去
         释放旧数组

扩容后： [ 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 | 11 | 空 | 空 | 空 | 空 ]
                                                                         ↑ 容量 15
```

如果你事先知道数据量很大，可以指定初始容量避免频繁扩容：

```java
List<String> list = new ArrayList<>(1000);  // 直接给 1000 容量
```

---

### 优缺点速记

```
✅ 查询快（O(1)）+ 尾部插入快（O(1)）+ 省内存
❌ 中间插入/删除慢（O(n)）+ 扩容有开销 + 线程不安全

→ 适合：读多写少、尾部追加、按索引查
→ 不适合：频繁中间插入删除、多线程并发
```

### 面试回答

> **面试官：说一下 ArrayList 的优缺点？**
>
> 优点是**查询快**，底层是数组，`get(i)` 是 O(1)；**尾部插入也快**，直接追加就行；而且比 LinkedList 省内存，不用存指针。
>
> 缺点是**中间插入和删除慢**，要移动后面的元素，平均 O(n)；**扩容有开销**，默认容量 10，满了要新建数组拷贝；另外**线程不安全**，多线程下需要用 `CopyOnWriteArrayList` 或者 `Collections.synchronizedList()`。

### 面试简答版

- 优点：查询 O(1)、尾部插入 O(1)、省内存、CPU 缓存友好
- 缺点：中间插入/删除 O(n)、扩容有拷贝开销、线程不安全、不能存基本类型
- 一句话：**读多写少、尾部操作多就用 ArrayList**

---

## 7. 如何实现数组和 List 之间的转换？

### 先讲场景

这是开发里**最常见的操作之一**，比如：

- 从数据库查出来的是一个 `List`，但你要调的方法参数是数组
- 或者你的数据是写死在数组里的，但你要传给一个接受 `List` 的方法

两个方向都需要掌握，而且各有各的坑。

---

### 方向一：List → 数组（用 `toArray()`）

```java
List<String> list = new ArrayList<>(List.of("A", "B", "C"));

// 方式一：返回 Object[]（不推荐）
Object[] arr1 = list.toArray();

// 方式二：传入指定类型的数组（推荐）
String[] arr2 = list.toArray(new String[0]);
```

**为什么推荐方式二？**

```java
Object[] arr = list.toArray();
// 返回的是 Object[]，不能强转成 String[]
String[] s = (String[]) arr;  // ❌ ClassCastException！
```

而 `toArray(new String[0])` 返回的就是 `String[]`，类型安全。

```java
String[] arr = list.toArray(new String[0]);  // ✅ 推荐
```

**那传 `new String[0]` 和 `new String[list.size()]` 哪个好？**

```java
// 写法一：传一个空数组
String[] arr1 = list.toArray(new String[0]);

// 写法二：传一个刚好够长的数组
String[] arr2 = list.toArray(new String[list.size()]);
```

JDK 里有个性能优化：如果传入的数组够大，就直接把元素放进去；如果不够大，就新建一个数组返回。

以前觉得写法二更快（省了创建空数组的开销），但 JDK 6 之后 `new String[0]` 做了优化，写法一反而更优——因为**空数组是常量，可以复用，而 `new String[list.size()]` 每次都要创建**。

```java
// 官方推荐写法（"java" 这个单词的由来）
String[] arr = list.toArray(new String[0]);
// ↑ 据说是因为 list.toArray(new String[0]) 首字母拼起来是 "ltns"，
// 读作 "litens"，后来简化为这个习惯
```

---

### 方向二：数组 → List

```java
String[] arr = {"A", "B", "C"};
```

#### 方式一：Arrays.asList()（⚠️ 有陷阱）

```java
List<String> list = Arrays.asList(arr);
```

**⚠️ 三个坑要记住：**

```java
// 坑 1：不能增删（只能改）
list.add("D");  // ❌ UnsupportedOperationException
list.remove("A"); // ❌ UnsupportedOperationException
list.set(0, "X"); // ✅ 可以改

// 坑 2：改数组会影响到 List，改 List 也会影响到数组
arr[0] = "X";
System.out.println(list.get(0));  // X ← 变了！

list.set(1, "Y");
System.out.println(arr[1]);  // Y ← 也变了！

// 坑 3：基本类型数组会出问题
int[] nums = {1, 2, 3};
List<int[]> list = Arrays.asList(nums);  // ❌ 把整个 int[] 当成了一个元素！
System.out.println(list.size());  // 1 ← 不是 3！
```

坑 3 的原因：`Arrays.asList(T...)` 接受泛型，`int` 是基本类型不是对象，所以整个 `int[]` 被当成了一个 `T`。改成 `Integer[]` 就好了。

#### 方式二：new ArrayList<>(Arrays.asList())（最稳妥）

```java
List<String> list = new ArrayList<>(Arrays.asList(arr));
```

**既没有"不能增删"的限制，也和原数组切断了关系**。改数组不影响 List，改 List 不影响数组。实际开发中最常用。

#### 方式三：List.of()（JDK 9+，不可变）

```java
List<String> list = List.of(arr);
```

和 `List.of("A", "B", "C")` 一样，返回**真正不可变**的 List，不能改、不能增、不能删。

#### 方式四：Collections.addAll()（另一种写法）

```java
List<String> list = new ArrayList<>();
Collections.addAll(list, arr);
```

---

### 完整转换总结

| 方向 | 方式 | 结果是否可变 | 和原数据是否关联 |
|:---|:---|:---:|:---:|
| List → 数组 | `list.toArray(new T[0])` | 数组就是新的 | 无关联 |
| 数组 → List | `Arrays.asList(arr)` | ❌ 不能增删 | ✅ 关联 |
| 数组 → List | `new ArrayList<>(Arrays.asList(arr))` | ✅ 完全可变 | ❌ 无关联 |
| 数组 → List | `List.of(arr)` | ❌ 完全不可变 | ❌ 无关联 |

### 面试回答

> **面试官：如何实现数组和 List 之间的转换？**
>
> 两个方向：
>
> List 转数组用 `list.toArray(new T[0])`，推荐传一个长度为 0 的数组，类型安全。
>
> 数组转 List 最常用的是 `Arrays.asList()`，但要注意它返回的 List 不能增删，而且修改数组会影响到 List。如果需要一个完全独立的可变 List，用 `new ArrayList<>(Arrays.asList(arr))`。JDK 9 之后也可以用 `List.of(arr)`，但返回的是不可变 List。
>
> 还有一个坑：`Arrays.asList()` 传基本类型数组（如 `int[]`）会把整个数组当成一个元素，记得用包装类型 `Integer[]`。

### 面试简答版

- List → 数组：`list.toArray(new T[0])`
- 数组 → List：`Arrays.asList(arr)`（不能增删，和数组关联）
- 数组 → 可变 List：`new ArrayList<>(Arrays.asList(arr))`
- 数组 → 不可变 List：`List.of(arr)`（JDK 9+）
- 注意：`Arrays.asList()` 传基本类型数组会出问题，要用包装类型

---

## 8. ArrayList 和 LinkedList 的区别是什么？

### 先讲场景

面试必问题，没有之一。

而且面试官不会只让你背区别表，他会追着问：

> "你刚才说 ArrayList 查询快，那是多快？LinkedList 呢？为什么？"

所以关键在于**理解底层结构**，而不是死记硬背。

---

### 一句话说清

```
ArrayList  = 数组（连续内存，有索引）
LinkedList = 双向链表（分散内存，靠指针连接）
```

这个底层结构的不同，导致了两者在所有方面的表现都不一样。

---

### 对比表

| 对比维度 | ArrayList | LinkedList |
|:---|:---|:---|
| **底层结构** | **动态数组**（Object[]） | **双向链表**（Node 节点） |
| **按索引查询** | **O(1)** ✅ 直接算地址 | **O(n)** ❌ 要遍历 |
| **尾部插入** | **O(1)** ✅ 直接追加 | **O(1)** ✅ 改尾指针 |
| **头部插入** | **O(n)** ❌ 要移位 | **O(1)** ✅ 改头指针 |
| **中间插入** | **O(n)** ❌ 要移位 | **O(n)** ❌ 要先找到位置 |
| **内存** | 省（只存数据+预留空间） | 费（每个节点多存前后指针） |
| **内存连续性** | ✅ 连续（CPU 缓存友好） | ❌ 不连续 |
| **RandomAccess** | ✅ 实现了 | ❌ 没实现 |
| **线程安全** | ❌ 都不安全 | ❌ 都不安全 |

---

### 核心区别详解

#### 1. 底层结构

```java
// ArrayList 的内存（连续）：
index: [ 0 | 1 | 2 | 3 | 4 | 5 |  预留  |  预留  ]
数据:   [ A | B | C | D | E |   |       |       ]

// LinkedList 的内存（分散，靠指针连起来）：
        ┌──────┐    ┌──────┐    ┌──────┐    ┌──────┐
        │prev  │←──→│prev  │←──→│prev  │←──→│prev  │
        │A     │    │B     │    │C     │    │D     │
        │next──│──→│next──│──→│next──│──→│next  │
        └──────┘    └──────┘    └──────┘    └──────┘
```

ArrayList 的每个元素只知道"我前面和后面还有元素"（因为是连续地址），但 LinkedList 的每个节点还要存 `prev` 和 `next` 两个指针——**这就是它更费内存的原因**。

#### 2. 按索引查询

```java
// ArrayList.get(3)
// 直接算：首地址 + 3 × 元素大小 → 一步到位
String s = arrayList.get(3);  // O(1) ✅

// LinkedList.get(3)
// 从头节点开始，next → next → next，走 3 步
String s = linkedList.get(3); // O(n) ❌
```

LinkedList 即使做了优化（根据 index 判断离头近还是离尾近，走步数少的），仍然是 O(n)。

#### 3. 头尾插入

```java
// ArrayList 尾部插入：直接放在数组末尾
list.add("X");  // O(1) ✅（不扩容的情况下）

// ArrayList 头部插入：所有元素后移一位
list.add(0, "X");  // O(n) ❌
// [ A | B | C | D | 空 ]
//   ↓
// [ 空 | A | B | C | D ]   ← 全部后移
//   ↓
// [ X | A | B | C | D ]   ← 头部放入 X

// LinkedList 头部插入：改头指针
linkedList.addFirst("X");  // O(1) ✅
// 新建节点 X，让 X.next 指向原来的头 A，头指针指向 X

// LinkedList 尾部插入：改尾指针
linkedList.addLast("X");   // O(1) ✅
```

#### 4. 中间插入

```java
// ArrayList 中间插入：后面的先移位，再插入
list.add(2, "X");  // O(n)

// LinkedList 中间插入：先遍历找到位置，再改指针
linkedList.add(2, "X");  // O(n)
```

**注意**：中间插入两者都是 O(n)，但原因不同：
- ArrayList 慢在**移位**（数据复制）
- LinkedList 慢在**查找**（遍历找位置）

#### 5. 内存占用

```java
// ArrayList：只存数据本身，外加一些预留空间
// 存 100 个 String，容量 100 → 占 100 个引用 + 少量数组对象开销

// LinkedList：每个节点要额外存 prev 和 next 两个指针
// 存 100 个 String → 100 个 Node 对象 + 200 个引用（prev + next）
// 占用内存大约是 ArrayList 的 2~3 倍
```

---

### 怎么选？

```
默认选 ArrayList。

除非你遇到的情况是：
  → 频繁在头部插入/删除（LinkedList 的 addFirst/removeFirst 是 O(1)）
  → 需要频繁做队列/双端队列操作
  
否则 ArrayList 在绝大多数场景下都优于 LinkedList。
```

```java
// 常见的错误选择
LinkedList<String> list = new LinkedList<>();  // ❌ 没必要
// 后面只调了 add() 和 get()，没有用到 addFirst/removeFirst 等链表特性

// 正确的选择
ArrayList<String> list = new ArrayList<>();     // ✅ 默认选这个
```

### 面试回答

> **面试官：ArrayList 和 LinkedList 的区别？**
>
> 底层结构完全不同：ArrayList 是动态数组，LinkedList 是双向链表。
>
> 这导致：ArrayList 按索引查询 O(1)，LinkedList 是 O(n)。尾部插入都是 O(1)，但头部插入 ArrayList 是 O(n)（要移位），LinkedList 是 O(1)（改指针）。
>
> 内存上，ArrayList 更省（只存数据），LinkedList 每个节点多存两个指针，是 ArrayList 的 2~3 倍。
>
> 实际开发中**默认选 ArrayList**，除非明确需要频繁头尾操作才考虑 LinkedList。

### 面试简答版

- ArrayList = 数组（连续内存，有索引）
- LinkedList = 双向链表（分散内存，指针连接）
- 查询：ArrayList O(1) vs LinkedList O(n)
- 头部插入：ArrayList O(n) vs LinkedList O(1)
- 中间插入：都是 O(n)，ArrayList 慢在移位，LinkedList 慢在查找
- 内存：ArrayList 省，LinkedList 多 2~3 倍
- **默认选 ArrayList，特殊情况才用 LinkedList**

---

## 9. ArrayList 和 Vector 的区别是什么？

### 先讲场景

面试官问完 ArrayList 和 LinkedList 的区别，经常会接着问：

> "那 ArrayList 和 Vector 呢？它俩底层都是数组，有什么区别？"

答案就四个字：**线程安全**。

---

### 一句话说清

```
ArrayList → 线程不安全，性能好
Vector    → 线程安全（方法加了 synchronized），性能差
```

两者底层**都是动态数组（Object[]）**，核心区别就是一个锁了，一个没锁。

---

### 对比表

| 对比维度 | ArrayList | Vector |
|:---|:---|:---|
| **出现时间** | JDK 1.2 | **JDK 1.0**（遗留类） |
| **线程安全** | ❌ 不安全 | ✅ 所有方法 synchronized |
| **性能** | ✅ 高 | ❌ 低（有锁开销） |
| **扩容倍数** | **1.5 倍** | **2 倍** |
| **扩容指定增量** | ❌ 不支持 | ✅ 可以设 capacityIncrement |
| **迭代器** | Iterator / ListIterator | Iterator / ListIterator + **Enumeration** |
| **是否推荐** | ✅ 首选 | ❌ 基本不用了 |

---

### 核心区别详解

#### 1. 线程安全——最根本的区别

```java
// Vector 几乎所有方法都加了 synchronized
public class Vector<E> {
    public synchronized boolean add(E e) { ... }
    public synchronized E get(int index) { ... }
    public synchronized E remove(int index) { ... }
    // ...
}
```

这意味着在多线程下 Vector 是安全的，但代价是**所有操作串行化**——一个线程在 add，其他线程想 add 就得等着。

```java
// 实际开发中，Vector 已经被 ConcurrentHashMap 的"List 版本"取代了：
// 想要线程安全 → 用 CopyOnWriteArrayList 或 Collections.synchronizedList()
```

**但注意**：Vector 的线程安全也只是"每个方法单独安全"，如果你做了复合操作（比如先判断再操作），仍然要自己加锁。

```java
// 即使 Vector 是线程安全的，这种写法仍然有问题
if (vector.size() > 0) {
    vector.get(0);  // 这两行之间，其他线程可能把元素删光了
}
```

#### 2. 扩容策略

```java
// ArrayList：不够时扩容 1.5 倍
// 容量 10 → 15 → 22 → 33 → 49 → ...
private int newCapacity(int minCapacity) {
    int oldCapacity = elementData.length;
    int newCapacity = oldCapacity + (oldCapacity >> 1);  // 右移1位 = 除以2
    // old + old/2 = 1.5 倍
}

// Vector：不够时扩容 2 倍
// 容量 10 → 20 → 40 → 80 → ...
protected int capacityIncrement;  // 可以自定义增量
private int newCapacity(int minCapacity) {
    int oldCapacity = elementData.length;
    int newCapacity = oldCapacity + ((capacityIncrement > 0) ?
                                     capacityIncrement : oldCapacity);
    // 如果没有设 increment，就扩容 2 倍
}
```

Vector 可以自定义每次扩容加多少：

```java
Vector<String> v = new Vector<>(10, 5);
// 初始容量 10，每次扩容加 5：10 → 15 → 20 → 25 → ...
```

#### 3. 历史地位

Vector 是 **JDK 1.0** 就有的遗留类，ArrayList 是 **JDK 1.2** 集合框架重构时才加入的。

```
JDK 1.0：Vector, Hashtable  ← 全加锁，性能差
JDK 1.2：ArrayList, HashMap  ← 不加锁，性能好 + Collections.synchronizedXxx()
JDK 1.5+：CopyOnWriteArrayList, ConcurrentHashMap  ← 真正的并发方案
```

### 现在怎么选？

```java
// 单线程 → 无脑 ArrayList
List<String> list = new ArrayList<>();

// 多线程 → 也别用 Vector，用更好的替代方案
// 读多写少 → CopyOnWriteArrayList
List<String> list = new CopyOnWriteArrayList<>();
// 或者手动包装
List<String> list = Collections.synchronizedList(new ArrayList<>());
```

**Vector 已经被淘汰了**，面试知道它的历史和区别就行，写代码别用。

### 面试回答

> **面试官：ArrayList 和 Vector 的区别？**
>
> 核心区别是**线程安全**：Vector 所有方法都加了 synchronized，线程安全但性能差；ArrayList 线程不安全但性能好。
>
> 其他区别：ArrayList 扩容 1.5 倍，Vector 扩容 2 倍（还可以自定义扩容增量）。Vector 是 JDK 1.0 的遗留类，ArrayList 是 JDK 1.2 集合框架引入的。
>
> 实际开发中 Vector 基本不用了。单线程用 ArrayList，多线程用 CopyOnWriteArrayList 或者 Collections.synchronizedList()。

### 面试简答版

- 线程安全：Vector ✅（synchronized），ArrayList ❌
- 性能：Vector 差，ArrayList 好
- 扩容：Vector 2 倍，ArrayList 1.5 倍（Vector 还能自定义增量）
- Vector 是 JDK 1.0 遗留类，**现在基本不用了**
- 多线程替代方案：`CopyOnWriteArrayList` 或 `Collections.synchronizedList()`


## 10. 插入数据时，ArrayList、LinkedList、Vector 谁速度较快？

### 先讲场景

这道题的陷阱在于：**答案不是固定的，要看插在哪**。

你问"谁快"，得先问"插在哪"——尾部、头部、还是中间？三种情况答案完全不一样。

---

### 先给结论

```text
尾部插入：ArrayList ≈ LinkedList  > Vector
         （O(1)）    （O(1)）     （O(1) + synchronized 锁开销）

头部插入：LinkedList  >>  ArrayList  >  Vector
         （O(1)）       （O(n)）      （O(n) + 锁开销）

中间插入：三者差距不大（都是 O(n)，但原因不同）
```

---

### 详细分析

#### 尾部插入

```java
// ArrayList：直接追加到数组末尾
list.add("X");
// [ A | B | C | D | _ ]    ← 有空位，直接放
// [ A | B | C | D | X ]    ✅ 一步到位

// 但如果满了，要先扩容（1.5 倍，拷贝全部元素）
// [ A | B | C | D | E | _ | _ | _ ]  ← 容量翻倍后再追加
```

- **ArrayList** ：不扩容时 O(1)，扩容时 O(n)。但扩容不频繁，平均还是 O(1)（均摊）
- **LinkedList** ：永远是 O(1)，改尾指针就行，不需要扩容
- **Vector** ：和 ArrayList 一样的数组结构，但每个方法都加了 `synchronized`，有**锁的获取和释放开销**

**三者差距**：ArrayList 和 LinkedList 差不多，Vector 明显慢一截（锁开销）。实测差别不大时 ArrayList 甚至会更快（CPU 缓存友好）。

#### 头部插入

```java
// ArrayList 头部插入：全部后移
list.add(0, "X");
// [ A | B | C | D | E ]
//   ↓ 全部后移
// [ _ | A | B | C | D | E ]
//   ↓
// [ X | A | B | C | D | E ]    ← 100 个元素就是 100 次移动

// LinkedList 头部插入：改指针
linkedList.addFirst("X");
// 新建节点 X，指向原头节点 A，头指针指向 X
// X → A → B → C → D → E        ← 不管多少个元素，一步搞定
```

**差距巨大**：`LinkedList.addFirst()` 是 O(1)，`ArrayList.add(0, x)` 是 O(n)。10000 个元素时差距上千倍。

#### 中间插入

```java
// ArrayList 中间插入：先移位，再放
list.add(2, "X");
// 移位 O(n)

// LinkedList 中间插入：先遍历找到位置，再改指针
linkedList.add(2, "X");
// 查找 O(n)
```

两者都是 O(n)，但：
- ArrayList 的 O(n) 是**数据复制**（连续内存，CPU 缓存友好）
- LinkedList 的 O(n) 是**指针遍历**（不连续内存，CPU 缓存不友好）

实际测试中，ArrayList 的中间插入**通常比 LinkedList 快**，因为内存连续、复制效率高，而 LinkedList 的节点分散在内存各处，遍历开销反而更大。

---

### 总结表

| 插入位置 | ArrayList | LinkedList | Vector | 胜出 |
|:---|:---|:---|:---|:---:|
| **尾部** | O(1) 均摊 | O(1) | O(1) + 锁 | ArrayList ≈ LinkedList |
| **头部** | O(n) | **O(1)** | O(n) + 锁 | **LinkedList** |
| **中间** | O(n)（数据复制） | O(n)（链表遍历） | O(n) + 锁 | ArrayList 通常稍快 |

### 面试回答

> **面试官：插入数据时，ArrayList、LinkedList、Vector 谁快？**
>
> 取决于插在哪。
>
> 尾部插入：ArrayList 和 LinkedList 都是 O(1)，速度差不多。Vector 因为加了 synchronized，会慢一些。
>
> 头部插入：LinkedList 是 O(1)，只需要改指针，远快于 ArrayList 和 Vector（它们要全部移位）。
>
> 中间插入：三者都是 O(n)，但实际测试 ArrayList 通常比 LinkedList 快，因为 ArrayList 的内存连续，复制数据比 LinkedList 的分散遍历效率高。
>
> 再补一句的话：Vector 不管插哪都最慢，因为 synchronized 锁有开销，而且它已经被淘汰了，面试归面试，写代码别用。

### 面试简答版

- 尾部：ArrayList ≈ LinkedList > Vector（Vector 有锁）
- 头部：LinkedList >>> ArrayList > Vector（LinkedList 是 O(1)）
- 中间：三者都是 O(n)，ArrayList 实际更快（内存连续，复制效率高）
- Vector 永远最慢（synchronized 锁开销）

## 11. 多线程场景下如何使用 ArrayList？

### 先讲场景

`ArrayList` 不是线程安全的。但多线程开发中你很可能需要"一个可以动态增长的数组"，怎么办？

直接看问题：

```java
List<String> list = new ArrayList<>();

// 两个线程同时往里面加数据
Thread t1 = new Thread(() -> {
    for (int i = 0; i < 1000; i++) list.add("t1-" + i);
});

Thread t2 = new Thread(() -> {
    for (int i = 0; i < 1000; i++) list.add("t2-" + i);
});

t1.start();
t2.start();
Thread.sleep(2000);

System.out.println(list.size());  // 预期 2000，实际可能 1800 多
```

跑了 2000 次 add，结果不到 2000。更糟的情况会直接抛 `ArrayIndexOutOfBoundsException`（因为 ArrayList 扩容时两个线程同时操作，数组状态不一致）。

---

### 三种解决方案

#### 方案一：Collections.synchronizedList()（最通用）

```java
List<String> list = Collections.synchronizedList(new ArrayList<>());
```

**原理**：给 ArrayList 套了一个壳子，每个方法都加了 `synchronized` 块。一次只有一个线程能操作。

```java
// 内部大致是这样
public void add(E e) {
    synchronized (mutex) {  // 锁住整个 List
        list.add(e);
    }
}
```

**优点**：简单，一包就行

**缺点**：
- 性能一般，所有操作串行
- **遍历时要手动加锁**（这个容易忘）

```java
List<String> list = Collections.synchronizedList(new ArrayList<>());

// ❌ 错误：遍历时没加锁
for (String s : list) {  // 可能抛 ConcurrentModificationException
    System.out.println(s);
}

// ✅ 正确：遍历时手动加锁
synchronized (list) {
    for (String s : list) {
        System.out.println(s);
    }
}
```

#### 方案二：CopyOnWriteArrayList（推荐，读多写少场景）

```java
List<String> list = new CopyOnWriteArrayList<>();
```

**原理**：读操作完全无锁，写操作时**复制一份新数组**，在副本上改，改完替换原数组引用。

```java
// 写的时候：
// 1. 复制整个数组
// 2. 在新数组上改
// 3. 把新数组赋给原引用

// 读的时候：直接读，不加锁，不阻塞
```

```java
CopyOnWriteArrayList<String> list = new CopyOnWriteArrayList<>();
list.add("A");
list.add("B");
list.add("C");

// 读：无锁
for (String s : list) {
    System.out.println(s);
}

// 写：内部加锁，但对你透明
list.add("D");
```

**优点**：读操作性能极高（无锁），遍历时不需要手动加锁，不会抛 `ConcurrentModificationException`

**缺点**：写操作有复制成本（数据量大时明显），不适合**写频繁**的场景

#### 方案三：手动加锁

```java
List<String> list = new ArrayList<>();

synchronized (someLock) {
    list.add("X");
}
```

不推荐，因为你得记住每一处都加锁，漏一处就出 bug。

---

### 三种方案对比

| 方案 | 读性能 | 写性能 | 遍历要加锁？ | 推荐场景 |
|:---|:---:|:---:|:---:|:---|
| `Collections.synchronizedList()` | 一般（有锁） | 一般（有锁） | ✅ 要 | 简单通用方案 |
| `CopyOnWriteArrayList` | **极高（无锁）** | 低（复制数组） | ❌ 不用 | **读多写少** |
| 手动加锁 | 取决于你怎么锁 | 取决于你怎么锁 | 取决于你 | 不推荐 |

### 面试回答

> **面试官：多线程场景下如何使用 ArrayList？**
>
> ArrayList 不是线程安全的，多线程下用主要有两种方案：
>
> 一是 `Collections.synchronizedList(new ArrayList<>())`，给 ArrayList 每个方法加锁，简单通用，但性能一般，而且遍历要手动加锁。
>
> 二是 `CopyOnWriteArrayList`，读无锁，写时复制数组，适合读多写少的场景。遍历不需要加锁，也不抛 ConcurrentModificationException。
>
> 如果写操作非常频繁，CopyOnWriteArrayList 的复制成本会很高，这时候就要根据实际情况评估了。

### 面试简答版

- ArrayList 多线程不安全（add 会丢数据，甚至数组越界）
- `Collections.synchronizedList()` → 方法级加锁，**遍历要手动加锁**
- `CopyOnWriteArrayList` → **读无锁**，写时复制，推荐读多写少场景
- 千万别手动加锁，容易漏

## 12. 为什么 ArrayList 的 elementData 用 transient 修饰？

### 先讲场景

如果你看过 ArrayList 源码，会发现 `elementData` 数组是这样声明的：

```java
public class ArrayList<E> {
    transient Object[] elementData;  // 为什么加 transient？
    private int size;
}
```

`transient` 的意思是：**这个字段不参与默认序列化**。

但问题是——`elementData` 是 ArrayList 存数据的核心数组，不让它序列化，那数据怎么保存？

这就引出了一个面试爱问的设计问题。

---

### 为什么不能直接序列化整个数组？

看 ArrayList 的内存布局：

```java
// 假设你 new ArrayList()，然后 add 了 3 个元素
// ArrayList 默认容量 10，所以实际内存是：
elementData = [ A | B | C | null | null | null | null | null | null | null ]
              ↑ 实际只有 3 个数据 ↑           7 个空位             ↑ 容量 10
```

如果直接用默认序列化，会把这 10 个位置（包括 7 个 null）全部写出去。

**这显然浪费空间和时间**——数据只有 3 个，却序列化了 10 个。

---

### ArrayList 怎么处理的？

ArrayList **自己重写了序列化方法**，只序列化实际有数据的那部分：

```java
// 自定义序列化：只写 size 个元素，不写空白位置
private void writeObject(java.io.ObjectOutputStream s) {
    s.defaultWriteObject();      // 先写非 transient 字段（size 等）
    s.writeInt(size);            // 写元素个数

    for (int i = 0; i < size; i++) {
        s.writeObject(elementData[i]);  // 只写 size 个有数据的元素
    }
}

// 自定义反序列化
private void readObject(java.io.ObjectInputStream s) {
    s.defaultReadObject();
    int size = s.readInt();      // 读元素个数

    elementData = new Object[size];  // 创建刚好大小的数组
    for (int i = 0; i < size; i++) {
        elementData[i] = s.readObject();  // 只读 size 个
    }
}
```

**这样一来：**

```
默认序列化（不加 transient）：序列化 10 个位置 → 写 10 个对象
自定义序列化（加 transient）：序列化 3 个元素 → 写 3 个对象

数据量大时差距更明显：
容量 10000，实际存 5000  → 自定义序列化省一半
```

---

### 用个比喻

> 你有一个能装 10 件衣服的行李箱（容量），但里面只放了 3 件衣服（size）。
>
> 默认序列化 = 把行李箱整个寄走，包括 7 个空位——浪费运费
> 自定义序列化 = 只把 3 件衣服拿出来打包寄走，到了再买新箱子装——省钱

### 面试回答

> **面试官：为什么 ArrayList 的 elementData 用 transient 修饰？**
>
> 因为 ArrayList 的底层数组 `elementData` 通常有**空闲容量**（capacity > size）。如果直接用默认序列化，会把所有空位（null）也序列化出去，浪费空间。
>
> ArrayList 重写了 `writeObject()` 和 `readObject()`，只序列化实际存储的 `size` 个元素，不序列化空位。用 `transient` 就是为了告诉 JVM"这个字段别用默认方式序列化，我自己来"。
>
> 这算是一个空间和性能上的优化。

### 面试简答版

- `transient` 让 `elementData` 不参与默认序列化
- 因为数组有**空闲容量**，默认序列化会把 null 也写出去，浪费
- ArrayList 自定义了 `writeObject()` / `readObject()`，只序列化 `size` 个有效元素
- 纯粹的空间/性能优化

## 13. List 和 Set 的区别

### 先讲场景

List 和 Set 都继承自 `Collection`，是 Java 集合里最常用的两个接口。

最根本的区别就两点：**能不能重复**、**有没有顺序**。

```java
// List：按顺序存，可以重复
List<String> list = new ArrayList<>();
list.add("A");
list.add("B");
list.add("A");       // ✅ 可以重复
System.out.println(list);  // [A, B, A] ← 保持插入顺序

// Set：不重复（自动去重）
Set<String> set = new HashSet<>();
set.add("A");
set.add("B");
set.add("A");        // ❌ 加不进去，Set 会自动去重
System.out.println(set);   // [A, B] ← 顺序不固定（HashSet）
```

---

### 对比表

| 对比维度 | List | Set |
|:---|:---|:---|
| **是否可重复** | ✅ 允许重复 | ❌ **自动去重** |
| **是否有序** | ✅ **按插入顺序** | ❌ 不一定（看实现） |
| **是否有索引** | ✅ 有（`get(i)`） | ❌ 没有 |
| **能否存 null** | ✅ 可以 | ✅ 可以（但 TreeSet 不行） |
| **常用实现** | ArrayList, LinkedList | HashSet, TreeSet, LinkedHashSet |
| **查询方式** | 按索引 / 遍历 | 只能遍历 |

---

### 关键区别详解

#### 1. 重复元素——最核心的区别

```java
// List：来者不拒，不管有没有重复都往里放
list.add("A");
list.add("A");
list.add("A");
System.out.println(list.size());  // 3

// Set：自动去重，重复的加不进去
set.add("A");
set.add("A");
set.add("A");
System.out.println(set.size());  // 1
```

#### 2. 顺序——List 保证，Set 不一定

```java
// List：存进去是什么顺序，取出来就是什么顺序
List<String> list = new ArrayList<>();
list.add("A");
list.add("B");
list.add("C");
// 遍历永远是 A → B → C ✅

// HashSet：不保证顺序
Set<String> set = new HashSet<>();
set.add("A");
set.add("B");
set.add("C");
// 遍历可能是 C → A → B ❌

// LinkedHashSet：能保持插入顺序
Set<String> linkedSet = new LinkedHashSet<>();
linkedSet.add("A");
linkedSet.add("B");
linkedSet.add("C");
// 遍历是 A → B → C ✅

// TreeSet：按排序顺序（不是插入顺序）
Set<String> treeSet = new TreeSet<>();
treeSet.add("C");
treeSet.add("A");
treeSet.add("B");
// 遍历是 A → B → C（按字母排序）
```

#### 3. 索引——List 有，Set 没有

```java
// List：通过索引快速定位
String s = list.get(2);  // ✅ O(1)

// Set：没有 get(index)
set.get(2);  // ❌ 编译报错，Set 没有 get() 方法
```

所以你想按位置取元素，只能用 List。Set 只能遍历或通过 `contains()` 判断是否存在。

#### 4. 判断重复的机制不同

```java
// List：用 equals() 判断两个元素是否相同
// 你可以存两个"内容一样"的对象，只要引用不同就行

// Set：用 hashCode() + equals() 判断
// HashSet 先比 hashCode，再比 equals
// 所以存自定义对象到 Set 时，记得重写 hashCode() 和 equals()
```

### 怎么选？

```text
需要保留插入顺序、允许重复、按索引取 → List（ArrayList）
只需要去重，不关心顺序           → Set（HashSet）
需要去重 + 保持插入顺序          → Set（LinkedHashSet）
需要去重 + 自动排序             → Set（TreeSet）
```

### 面试回答

> **面试官：List 和 Set 的区别？**
>
> 核心区别两个：List 允许重复，Set 自动去重；List 按插入顺序存储，Set 不一定。
>
> 另外 List 有索引可以 `get(i)`，Set 没有索引只能遍历。
>
> 选型上：需要重复、需要按位置取 → List；需要去重 → Set。Set 的不同实现还分无序（HashSet）、保持插入顺序（LinkedHashSet）、自动排序（TreeSet）。

### 面试简答版

- 重复：List ✅ 允许，Set ❌ 自动去重
- 顺序：List ✅ 按插入顺序，Set 看实现（HashSet 无序、LinkedHashSet 有序、TreeSet 排序）
- 索引：List 有，Set 没有
- 判断重复：List 用 equals，Set 用 hashCode + equals
- 选型：要重复/索引选 List，要去重选 Set
