# Java 集合

## 1. Java 集合框架怎么理解？

### 核心理解
- Java 集合本质上就是一套“装数据 + 管数据 + 处理数据”的容器体系。
- 它主要分成两大支系：
  - `Collection`：存单个元素
  - `Map`：存键值对
- `Collection` 下面最核心的 3 个子接口是：
  - `List`
  - `Set`
  - `Queue`

### 一句话总结
- `Collection` 管“一个一个元素”。
- `Map` 管“key-value 键值对”。

### 结构速记
| 体系 | 典型接口 | 典型实现类 |
|---|---|---|
| 单列集合 | `List` | `ArrayList`、`LinkedList`、`Vector` |
| 单列集合 | `Set` | `HashSet`、`LinkedHashSet`、`TreeSet` |
| 单列集合 | `Queue` / `Deque` | `PriorityQueue`、`ArrayDeque`、`LinkedList` |
| 双列集合 | `Map` | `HashMap`、`LinkedHashMap`、`TreeMap`、`Hashtable`、`ConcurrentHashMap` |

## 2. List、Set、Queue、Map 四者有什么区别？

### 核心理解
- `List`
  - 有序
  - 可重复
  - 适合按下标访问、按顺序存储
- `Set`
  - 不可重复
  - 一般不强调下标
  - 适合做去重
- `Queue`
  - 强调排队、先进先出或特定优先级规则
  - 常用于任务队列、消息队列、缓冲队列
- `Map`
  - 存键值对
  - 通过 key 找 value
  - key 不可重复，value 可以重复

### 一句话总结
- `List`：有序可重复
- `Set`：无重复
- `Queue`：排队规则
- `Map`：键值映射

### 简单代码示例
```java
import java.util.*;

public class Demo {
    public static void main(String[] args) {
        List<String> list = new ArrayList<>();
        list.add("A");
        list.add("A");

        Set<String> set = new HashSet<>();
        set.add("A");
        set.add("A");

        Queue<String> queue = new LinkedList<>();
        queue.offer("first");
        queue.offer("second");

        Map<Integer, String> map = new HashMap<>();
        map.put(1, "Tom");
        map.put(1, "Jack");

        System.out.println(list);
        System.out.println(set);
        System.out.println(queue.poll());
        System.out.println(map);
    }
}
```

输出结果：
```text
[A, A]
[A]
first
{1=Jack}
```

这里可以这样理解：
- `List` 保留重复元素。
- `Set` 自动去重。
- `Queue` 先入先出。
- `Map` 相同 key 会覆盖旧 value。

## 3. 集合怎么选？

### 一个面试里很实用的选型思路
- 只想存值，并且允许重复：优先 `List`
- 只想存值，并且要求不重复：优先 `Set`
- 需要按 key 查值：用 `Map`
- 需要保持插入顺序：
  - `List` 天然支持
  - `Set` 选 `LinkedHashSet`
  - `Map` 选 `LinkedHashMap`
- 需要排序：
  - `Set` 选 `TreeSet`
  - `Map` 选 `TreeMap`
- 需要高频随机访问：
  - `List` 优先 `ArrayList`
- 需要头尾插入删除更频繁：
  - 双端队列优先考虑 `ArrayDeque`
  - 如果明确是链式场景才考虑 `LinkedList`
- 需要线程安全：
  - `Map` 优先 `ConcurrentHashMap`
  - 不建议优先用老的 `Hashtable`、`Vector`

### 一句话总结
- 查得多用数组结构，去重用 `Set`，映射用 `Map`，排序用 `Tree` 系列，并发优先专用并发集合。

## 4. Array 和 ArrayList 有什么区别？

### 核心理解
- 数组 `Array`
  - 长度固定
  - 可以存基本类型
  - 只能靠下标访问
- `ArrayList`
  - 底层是动态数组
  - 长度可扩容
  - 只能存对象，基本类型要用包装类
  - 提供丰富的 API，比如 `add`、`remove`、`set`

### 一句话总结
- 数组更原始、更固定。
- `ArrayList` 更灵活、更适合业务开发。

### 简单代码示例
```java
import java.util.ArrayList;
import java.util.Arrays;

public class Demo {
    public static void main(String[] args) {
        String[] arr = {"a", "b"};
        arr[0] = "x";
        System.out.println(Arrays.toString(arr));

        ArrayList<String> list = new ArrayList<>();
        list.add("a");
        list.add("b");
        list.add("c");
        System.out.println(list);
    }
}
```

输出结果：
```text
[x, b]
[a, b, c]
```

这里可以这样理解：
- 数组一开始长度就定死了。
- `ArrayList` 可以继续往后追加元素，更适合不确定数量的数据。

## 5. ArrayList 怎么理解？它适合什么场景？

### 核心理解
- `ArrayList` 是 `List` 最常用的实现类。
- 底层是 `Object[]` 动态数组。
- 它最擅长的是：
  - 随机访问
  - 尾部追加
  - 遍历

### 常见性能结论
- 按下标读：`O(1)`
- 尾部插入：通常 `O(1)`，扩容时会退化
- 头部插入 / 中间插入 / 删除：`O(n)`，因为要搬移元素

### 为什么随机访问快？
- 因为数组天然支持通过下标直接定位。
- 这就是 `ArrayList` 比 `LinkedList` 更适合“查得多”的本质原因。

### 面试高频补充
- `ArrayList` 允许存 `null`
- 线程不安全
- JDK 8 中第一次真正添加元素时才会分配默认容量

### 简单代码示例
```java
import java.util.ArrayList;

public class Demo {
    public static void main(String[] args) {
        ArrayList<Integer> list = new ArrayList<>();
        list.add(10);
        list.add(20);
        list.add(1, 15);

        System.out.println(list);
        System.out.println(list.get(2));
    }
}
```

输出结果：
```text
[10, 15, 20]
20
```

## 6. ArrayList 和 LinkedList 怎么区分？

### 核心理解
- `ArrayList`
  - 底层是动态数组
  - 查询快
  - 中间插入删除慢
- `LinkedList`
  - 底层是双向链表
  - 中间节点插入删除理论上更方便
  - 但按下标访问慢

### 一句话总结
- 查得多：`ArrayList`
- 频繁头尾操作、队列/双端队列语义明显：再考虑 `LinkedList`

### 你要特别注意的一点
- 很多人背成“增删用 LinkedList，查询用 ArrayList”，这句话太粗。
- 因为 `LinkedList` 找到节点本身也要时间。
- 所以在真实业务里，`ArrayList` 的使用频率通常远高于 `LinkedList`。

### 简单代码示例
```java
import java.util.ArrayList;
import java.util.LinkedList;
import java.util.List;

public class Demo {
    public static void main(String[] args) {
        List<String> a = new ArrayList<>();
        a.add("A");
        a.add("B");
        System.out.println(a.get(1));

        List<String> b = new LinkedList<>();
        b.add("A");
        b.add("B");
        System.out.println(b.get(1));
    }
}
```

输出结果：
```text
B
B
```

这里结果一样，但底层访问方式不同：
- `ArrayList` 直接按下标算位置
- `LinkedList` 要沿链表找节点

## 7. Set 体系怎么理解？

### 核心理解
- `Set` 的核心特点就是：不允许重复元素。
- 常见实现类主要看 3 个：
  - `HashSet`
  - `LinkedHashSet`
  - `TreeSet`

### 三者区别
#### 1）HashSet
- 无序
- 去重
- 底层基于 `HashMap`
- 查找性能通常不错

#### 2）LinkedHashSet
- 在 `HashSet` 基础上维护插入顺序
- 既去重，又能保留插入顺序

#### 3）TreeSet
- 自动排序
- 底层是红黑树
- 适合有序去重场景

### 一句话总结
- 去重但不关心顺序：`HashSet`
- 去重且保留插入顺序：`LinkedHashSet`
- 去重且自动排序：`TreeSet`

### 简单代码示例
```java
import java.util.*;

public class Demo {
    public static void main(String[] args) {
        Set<Integer> hashSet = new HashSet<>(Arrays.asList(3, 1, 2, 2));
        Set<Integer> linkedHashSet = new LinkedHashSet<>(Arrays.asList(3, 1, 2, 2));
        Set<Integer> treeSet = new TreeSet<>(Arrays.asList(3, 1, 2, 2));

        System.out.println(hashSet);
        System.out.println(linkedHashSet);
        System.out.println(treeSet);
    }
}
```

输出结果（`HashSet` 顺序不固定）：
```text
[1, 2, 3]
[3, 1, 2]
[1, 2, 3]
```

## 8. HashSet 为什么能去重？

### 核心理解
- `HashSet` 底层其实是 `HashMap`。
- 加元素时会先看 `hashCode()`
- 如果哈希值冲突，再进一步看 `equals()`
- 只有“哈希相同且 equals 也相同”，才会认为是重复元素

### 一句话总结
- `HashSet` 去重靠的是：`hashCode() + equals()`

### 简单代码示例
```java
import java.util.HashSet;
import java.util.Objects;
import java.util.Set;

class User {
    String name;

    User(String name) {
        this.name = name;
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof User)) return false;
        User user = (User) o;
        return Objects.equals(name, user.name);
    }

    @Override
    public int hashCode() {
        return Objects.hash(name);
    }
}

public class Demo {
    public static void main(String[] args) {
        Set<User> set = new HashSet<>();
        set.add(new User("Tom"));
        set.add(new User("Tom"));

        System.out.println(set.size());
    }
}
```

输出结果：
```text
1
```

这里可以这样理解：
- 两个 `User("Tom")` 被认为逻辑相等。
- 所以最终只保留一个。

## 9. Queue / Deque 体系怎么理解？

### 核心理解
- `Queue` 主要强调“排队处理”
- 最常见的规则是 FIFO：先进先出
- `Deque` 是双端队列，可以两头进、两头出

### 常见实现类
#### 1）LinkedList
- 既能当 `List`
- 也能当 `Queue` / `Deque`
- 但真实开发里，单纯做队列时不一定是首选

#### 2）ArrayDeque
- 基于可扩容数组
- 通常比 `LinkedList` 更适合实现栈和双端队列
- 不允许放 `null`

#### 3）PriorityQueue
- 不是先进先出
- 而是按优先级出队
- 底层是堆

### 一句话总结
- 普通队列/双端队列：优先考虑 `ArrayDeque`
- 优先级队列：`PriorityQueue`

### 简单代码示例
```java
import java.util.ArrayDeque;
import java.util.PriorityQueue;
import java.util.Queue;

public class Demo {
    public static void main(String[] args) {
        Queue<String> q = new ArrayDeque<>();
        q.offer("A");
        q.offer("B");
        System.out.println(q.poll());

        PriorityQueue<Integer> pq = new PriorityQueue<>();
        pq.offer(5);
        pq.offer(1);
        pq.offer(3);
        System.out.println(pq.poll());
    }
}
```

输出结果：
```text
A
1
```

这里可以这样理解：
- 普通队列先入先出，所以先取到 `A`
- `PriorityQueue` 默认小顶堆，所以先取到最小值 `1`

## 10. Map 体系怎么理解？

### 核心理解
- `Map` 用来存键值对：`key -> value`
- key 不可重复
- value 可以重复
- 最常考的实现类：
  - `HashMap`
  - `LinkedHashMap`
  - `TreeMap`
  - `Hashtable`
  - `ConcurrentHashMap`

### 一句话总结
- `HashMap` 最常用
- `LinkedHashMap` 保序
- `TreeMap` 排序
- `ConcurrentHashMap` 并发安全

## 11. HashMap 怎么理解？为什么它最常用？

### 核心理解
- `HashMap` 是最常用的 `Map` 实现。
- JDK 8 里底层是：
  - 数组
  - 链表
  - 红黑树
- 它的目标就是：尽量快地通过 key 找到 value。

### 一个直观理解
- 数组是大格子
- 每个 key 通过哈希运算落到某个格子
- 冲突时先挂链表
- 冲突严重时再转红黑树

### JDK 8 的关键点
- 数组长度通常保持 2 的幂次方
- 冲突链表长度达到阈值时，可能树化
- 但如果数组还太小，会优先扩容而不是立刻树化

### 为什么长度通常是 2 的幂？
- 因为这样 `(n - 1) & hash` 可以高效替代取模运算
- 同时有利于数据分布更均匀、扩容迁移更高效

### 一句话总结
- `HashMap` 的本质就是：用哈希定位桶位，用链表/红黑树处理冲突

## 12. HashMap、LinkedHashMap、TreeMap、Hashtable、ConcurrentHashMap 怎么区分？

### 1）HashMap
- 非线程安全
- 允许一个 `null` key，多个 `null` value
- 最常用

### 2）LinkedHashMap
- 继承 `HashMap`
- 在 `HashMap` 基础上维护双向链表
- 可以保留插入顺序，某些场景还能实现访问顺序

### 3）TreeMap
- 底层红黑树
- key 自动有序
- 查找、插入、删除通常是 `O(log n)`

### 4）Hashtable
- 线程安全
- 不允许 `null` key / value
- 老旧，实际开发一般不推荐优先使用

### 5）ConcurrentHashMap
- 专门为并发场景设计
- 线程安全
- 比 `Hashtable` 更适合现代并发开发

### 一句话总结
- 默认首选 `HashMap`
- 要保序用 `LinkedHashMap`
- 要排序用 `TreeMap`
- 要并发安全用 `ConcurrentHashMap`

## 13. HashMap 和 Hashtable 的区别是什么？

### 核心理解
- `HashMap`
  - 非线程安全
  - 效率更高
  - 允许 `null` key 和 `null` value
- `Hashtable`
  - 线程安全
  - 方法大量使用 `synchronized`
  - 不允许 `null`
  - 现在基本不推荐作为首选

### 一句话总结
- 老代码里可能看到 `Hashtable`
- 新代码里并发安全优先考虑 `ConcurrentHashMap`

## 14. HashMap 和 TreeMap 的区别是什么？

### 核心理解
- `HashMap`
  - 不保证顺序
  - 重点是快速存取
- `TreeMap`
  - 按 key 排序
  - 底层红黑树
  - 还支持很多范围查找、最近键查找

### 一句话总结
- 快速查找优先 `HashMap`
- 有序键和范围操作优先 `TreeMap`

### 简单代码示例
```java
import java.util.HashMap;
import java.util.Map;
import java.util.TreeMap;

public class Demo {
    public static void main(String[] args) {
        Map<Integer, String> hashMap = new HashMap<>();
        hashMap.put(3, "c");
        hashMap.put(1, "a");
        hashMap.put(2, "b");

        Map<Integer, String> treeMap = new TreeMap<>();
        treeMap.put(3, "c");
        treeMap.put(1, "a");
        treeMap.put(2, "b");

        System.out.println(hashMap);
        System.out.println(treeMap);
    }
}
```

输出结果（`HashMap` 顺序不固定）：
```text
{1=a, 2=b, 3=c}
{1=a, 2=b, 3=c}
```

补充说明：
- 这里 `HashMap` 这一轮输出恰好看起来有序，不代表它保证有序。
- `TreeMap` 才是真正按 key 排序。

## 15. Collection 和 Collections 有什么区别？

### 核心理解
- `Collection`
  - 是集合体系的顶层接口之一
  - 代表一组元素
- `Collections`
  - 是工具类
  - 提供排序、查找、同步包装、不可变包装等静态方法

### 一句话总结
- `Collection` 是接口
- `Collections` 是工具类

### 简单代码示例
```java
import java.util.ArrayList;
import java.util.Collections;
import java.util.List;

public class Demo {
    public static void main(String[] args) {
        List<Integer> list = new ArrayList<>();
        list.add(3);
        list.add(1);
        list.add(2);

        Collections.sort(list);
        System.out.println(list);
    }
}
```

输出结果：
```text
[1, 2, 3]
```

## 16. 这一章最该记住什么？

### 你可以先记这 8 句话
- 集合两大体系：`Collection` 和 `Map`
- `List` 有序可重复
- `Set` 去重
- `Queue` 强调排队规则
- `Map` 存键值对
- `ArrayList` 最常用，底层动态数组
- `HashMap` 最常用，底层数组 + 链表 / 红黑树
- 并发安全优先考虑 `ConcurrentHashMap`

### 面试高频误区
- 不要把 `LinkedList` 背成“绝对比 `ArrayList` 更适合增删”
- 不要把 `HashMap` 说成“完全无序”之后就什么都不解释，重点是它“不保证顺序”
- 不要说 `Hashtable` 还是主流线程安全 `Map`，现代开发更多是 `ConcurrentHashMap`
