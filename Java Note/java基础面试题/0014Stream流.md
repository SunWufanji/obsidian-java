# Stream流

## 1. 什么是 Stream 流？为什么要用它？

#### 先给结论
- `Stream` 是 Java 8 引入的一套“面向数据处理”的 API。
- 它不是用来存数据的，而是用来对数据做“筛选、转换、排序、统计、收集”等操作的。
- 你可以把它理解成一条流水线：数据从源头进来，经过一连串处理，最后得到结果。

#### 核心解释
- `Collection` 更关注“存什么”。
- `Stream` 更关注“怎么处理这些数据”。
- 以前我们做集合操作，常常要手写很多 `for` 循环、`if` 判断、临时集合。
- `Stream` 的价值就是把这些“遍历细节”收起来，让代码更聚焦在业务逻辑本身。
- 它特别适合这种场景：
  - 从一堆数据中筛选符合条件的元素
  - 对数据做映射转换
  - 统计个数、求最大值、分组、汇总

#### 简单代码示例
```java
import java.util.Arrays;
import java.util.List;

public class Demo {
    public static void main(String[] args) {
        List<String> list = Arrays.asList("张无忌", "周芷若", "张三", "张三丰");

        list.stream()
                .filter(name -> name.startsWith("张"))
                .forEach(System.out::println);
    }
}
```

#### 输出结果
```text
张无忌
张三
张三丰
```

#### 输出 / 现象解释
- `stream()` 先把集合变成流。
- `filter(...)` 负责筛选出姓“张”的元素。
- `forEach(...)` 负责最终输出。
- 整个过程非常像“源数据 -> 过滤 -> 输出”的流水线。

#### 面试易错点 / 补充点
- `Stream` 不是 `InputStream` / `OutputStream` 那种 IO 流，这两个“流”不是一回事。
- `Stream` 不是数据结构，不负责存储元素。
- 它更像“对一批数据做批量操作的声明式写法”。

#### 面试回答模板
- Stream 是 Java 8 引入的一套聚合操作 API，本质上是对数据源进行流水线式处理。它不存储数据，而是把集合、数组等数据源经过过滤、映射、排序、归约、收集等步骤处理后得到结果。相比传统 `for` 循环，它更强调“我要做什么”，代码表达力更强。

#### 面试简答版
- Stream 不存数据，只处理数据。
- 本质：流水线式的集合/数组聚合操作 API。

## 2. Stream 和 Collection 有什么区别？

#### 先给结论
- `Collection` 是数据容器，负责存储和管理元素。
- `Stream` 是数据处理管道，负责计算和转换元素。
- 两者的关注点完全不同。

#### 核心解释
##### 1）是否存储数据不同
- `Collection` 会真正把元素保存起来。
- `Stream` 不保存元素，它只是从数据源“搬运并处理”元素。

##### 2）是否可重复使用不同
- `Collection` 可以反复遍历。
- `Stream` 一般只能消费一次，用完就结束。

##### 3）操作目标不同
- `Collection` 主要是增删改查。
- `Stream` 主要是过滤、映射、排序、聚合、分组、收集。

##### 4）执行方式不同
- `Collection` 更偏立即操作。
- `Stream` 很多中间操作是惰性执行的，等终止操作来了才真正跑起来。

#### 简单代码示例
```java
import java.util.Arrays;
import java.util.List;

public class Demo {
    public static void main(String[] args) {
        List<Integer> list = Arrays.asList(1, 2, 3, 4);

        long count = list.stream().filter(x -> x > 2).count();

        System.out.println(list);
        System.out.println(count);
    }
}
```

#### 输出结果
```text
[1, 2, 3, 4]
2
```

#### 输出 / 现象解释
- `list` 作为数据源没有被改掉。
- `Stream` 只是基于这个集合做了一次“筛选 + 计数”的计算。

#### 面试易错点 / 补充点
- 不要把 `Stream` 说成“新集合”。
- 它可以产生新结果，但本身不是用来装元素的容器。

#### 面试回答模板
- Collection 和 Stream 的区别在于：Collection 是用来存储和管理数据的容器，而 Stream 是用来对数据做聚合处理的管道。Collection 可以反复遍历，Stream 通常只能消费一次；Collection 更关注元素管理，Stream 更关注过滤、映射、排序和统计等计算过程。

#### 面试简答版
- Collection：存数据。
- Stream：处理数据。
- Collection 可重复遍历，Stream 通常一次性消费。

## 3. Stream 的执行流程是什么？什么是中间操作和终止操作？

#### 先给结论
- 一个完整的 Stream 流水线一般由 3 部分组成：
  - 数据源（source）
  - 中间操作（intermediate operations）
  - 终止操作（terminal operation）
- 没有终止操作，前面的中间操作通常不会真正执行。

#### 核心解释
##### 1）数据源
- 可以是集合、数组、静态工厂方法、文件行流等。

##### 2）中间操作
- 比如 `filter`、`map`、`sorted`、`distinct`、`limit`
- 特点是：返回一个新的流，可以继续往后接。

##### 3）终止操作
- 比如 `forEach`、`count`、`collect`、`reduce`、`findFirst`
- 特点是：触发真正执行，并产生最终结果或副作用。

#### 简单代码示例
```java
import java.util.Arrays;
import java.util.List;

public class Demo {
    public static void main(String[] args) {
        List<Integer> list = Arrays.asList(1, 2, 3, 4, 5);

        long count = list.stream()
                .filter(x -> x > 2)
                .map(x -> x * 10)
                .count();

        System.out.println(count);
    }
}
```

#### 输出结果
```text
3
```

#### 输出 / 现象解释
- `filter(x -> x > 2)` 是中间操作。
- `map(x -> x * 10)` 也是中间操作。
- `count()` 是终止操作。
- 只有当 `count()` 出现时，前面的流水线才会真正开始跑。

#### 面试易错点 / 补充点
- 这题一定要说“惰性执行”。
- 中间操作不是立刻把结果算出来，而是先把处理规则挂在线路上。
- 一旦终止操作执行，整条流水线就会被触发。

#### 面试回答模板
- Stream 的执行流程是：先有数据源，再串联零个或多个中间操作，最后由一个终止操作触发执行。中间操作会返回新的流，通常是惰性执行；终止操作才会真正消费流并产生结果，比如 `count`、`collect` 或 `forEach`。

#### 面试简答版
- 流程：数据源 -> 中间操作 -> 终止操作。
- 中间操作惰性执行，终止操作负责触发执行。

## 4. Stream 怎么创建？

#### 先给结论
- 最常见的 4 种创建方式：
  - 集合：`list.stream()`
  - 数组：`Arrays.stream(arr)`
  - 静态工厂：`Stream.of(...)`
  - 生成流：`Stream.iterate(...)`、`Stream.generate(...)`

#### 核心解释
##### 1）集合创建
- 最常见，开发里也最实用。

##### 2）数组创建
- 适合已有数组数据。

##### 3）`Stream.of`
- 适合少量固定元素快速创建。

##### 4）`iterate` / `generate`
- 适合创建无限流或按规则生成的数据流。
- 这类流通常要配合 `limit()`，否则可能无限运行。

#### 简单代码示例
```java
import java.util.Arrays;
import java.util.List;
import java.util.stream.Stream;

public class Demo {
    public static void main(String[] args) {
        List<String> list = Arrays.asList("a", "b", "c");
        list.stream().forEach(System.out::println);

        Arrays.stream(new int[]{1, 2, 3}).forEach(System.out::println);

        Stream.of("x", "y").forEach(System.out::println);

        Stream.iterate(0, x -> x + 2)
                .limit(3)
                .forEach(System.out::println);
    }
}
```

#### 输出结果
```text
a
b
c
1
2
3
x
y
0
2
4
```

#### 输出 / 现象解释
- 前三种都是“直接把现有数据变成流”。
- `iterate` 是“按规则不断生成数据”的流，所以需要 `limit(3)` 截断。

#### 面试易错点 / 补充点
- `Stream.iterate()` 和 `generate()` 很容易造成无限流，要配合 `limit()`。
- 除了对象流，还有 `IntStream`、`LongStream`、`DoubleStream` 这些基本类型流。

#### 面试回答模板
- Stream 最常见的创建方式包括：通过集合的 `stream()` 方法、通过 `Arrays.stream()` 处理数组、通过 `Stream.of()` 创建固定元素流，以及通过 `Stream.iterate()`、`Stream.generate()` 创建按规则生成的流。实际开发中最常见的是集合流和数组流。

#### 面试简答版
- 常见创建方式：集合、数组、`Stream.of`、`iterate/generate`。

## 5. Stream 常见中间操作有哪些？

#### 先给结论
- 面试和开发里最常见的中间操作有：
  - `filter`
  - `map`
  - `flatMap`
  - `sorted`
  - `distinct`
  - `limit`
  - `skip`

#### 核心解释
##### 1）`filter`
- 过滤，保留满足条件的元素。

##### 2）`map`
- 一对一映射，把元素转换成另一种形式。

##### 3）`flatMap`
- 打平，把“流里的流”展开成一个流。

##### 4）`sorted`
- 排序，可以自然排序，也可以自定义比较器。

##### 5）`distinct`
- 去重，底层依赖 `equals()` 和 `hashCode()`。

##### 6）`limit`
- 截取前 N 个。

##### 7）`skip`
- 跳过前 N 个。

#### 简单代码示例
```java
import java.util.Arrays;
import java.util.List;
import java.util.stream.Collectors;

public class Demo {
    public static void main(String[] args) {
        List<Integer> list = Arrays.asList(5, 3, 3, 1, 2, 6);

        List<Integer> result = list.stream()
                .filter(x -> x > 1)
                .distinct()
                .sorted()
                .skip(1)
                .limit(3)
                .collect(Collectors.toList());

        System.out.println(result);
    }
}
```

#### 输出结果
```text
[3, 5, 6]
```

#### 输出 / 现象解释
- `filter` 保留大于 1 的元素
- `distinct` 去重
- `sorted` 升序排序
- `skip(1)` 跳过第一个
- `limit(3)` 取接下来的 3 个

#### 面试易错点 / 补充点
- 中间操作返回的还是流，不是最终结果。
- `distinct()` 去重效果依赖对象的 `equals()` / `hashCode()`，自定义对象面试里经常追问这个点。

#### 面试回答模板
- Stream 常见中间操作包括 `filter`、`map`、`flatMap`、`sorted`、`distinct`、`limit`、`skip` 等。它们都会返回新的流，本身通常不会立刻执行，而是等终止操作触发后统一完成整条流水线的计算。

#### 面试简答版
- 常见中间操作：过滤、映射、打平、排序、去重、截取、跳过。

## 6. Stream 常见终止操作有哪些？

#### 先给结论
- 常见终止操作主要有：
  - `forEach`
  - `count`
  - `collect`
  - `reduce`
  - `findFirst`
  - `findAny`
  - `anyMatch` / `allMatch` / `noneMatch`
  - `max` / `min`

#### 核心解释
##### 1）遍历类
- `forEach`：逐个处理元素

##### 2）查找匹配类
- `findFirst`：找第一个
- `findAny`：找任意一个，并行流里更常见
- `anyMatch` / `allMatch` / `noneMatch`：做条件匹配

##### 3）统计聚合类
- `count`：计数
- `max` / `min`：最大最小
- `reduce`：规约成一个值

##### 4）收集类
- `collect`：把结果收集成 `List`、`Set`、`Map`，或者做分组统计

#### 简单代码示例
```java
import java.util.Arrays;
import java.util.List;
import java.util.Optional;
import java.util.stream.Collectors;

public class Demo {
    public static void main(String[] args) {
        List<Integer> list = Arrays.asList(1, 2, 3, 4, 5);

        long count = list.stream().filter(x -> x > 2).count();
        Optional<Integer> first = list.stream().filter(x -> x > 2).findFirst();
        List<Integer> result = list.stream().map(x -> x * 10).collect(Collectors.toList());

        System.out.println(count);
        System.out.println(first.get());
        System.out.println(result);
    }
}
```

#### 输出结果
```text
3
3
[10, 20, 30, 40, 50]
```

#### 输出 / 现象解释
- `count()` 返回满足条件的元素个数
- `findFirst()` 返回第一个匹配元素，类型通常是 `Optional`
- `collect(...)` 把流的结果收集回集合

#### 面试易错点 / 补充点
- 终止操作执行后，流通常就不能再用了。
- `findFirst()`、`findAny()`、`max()` 这类结果很多时候返回的是 `Optional`，面试里要会解释原因：因为可能没有值。

#### 面试回答模板
- Stream 的终止操作负责真正触发流水线执行，常见的有遍历类的 `forEach`，统计类的 `count`、`max`、`min`，查找匹配类的 `findFirst`、`findAny`、`anyMatch`，以及收集类的 `collect` 和规约类的 `reduce`。终止操作执行后，流一般就不能再次复用。

#### 面试简答版
- 常见终止操作：`forEach`、`count`、`collect`、`reduce`、`findFirst`、`match`、`max/min`。

## 7. map 和 flatMap 有什么区别？

#### 先给结论
- `map` 是“一对一映射”。
- `flatMap` 是“先映射，再打平”。
- 这题面试里最重要的是理解“打平”两个字。

#### 核心解释
##### 1）`map`
- 每个元素映射成一个新元素。
- 结果还是“一层结构”。

##### 2）`flatMap`
- 每个元素可以映射成一个流或集合。
- 然后把这些子流、子集合全部拍平合并成一个总流。

#### 简单代码示例
```java
import java.util.Arrays;
import java.util.List;
import java.util.stream.Collectors;

public class Demo {
    public static void main(String[] args) {
        List<String> list = Arrays.asList("a,b", "c,d");

        List<String> mapResult = list.stream()
                .map(s -> s.split(",")[0])
                .collect(Collectors.toList());

        List<String> flatResult = list.stream()
                .flatMap(s -> Arrays.stream(s.split(",")))
                .collect(Collectors.toList());

        System.out.println(mapResult);
        System.out.println(flatResult);
    }
}
```

#### 输出结果
```text
[a, c]
[a, b, c, d]
```

#### 输出 / 现象解释
- `map` 这里只取了每段字符串拆分后的第一个元素，所以得到 `[a, c]`
- `flatMap` 则把每个字符串拆成一个小流，再把多个小流打平成一个大流，所以得到 `[a, b, c, d]`

#### 面试易错点 / 补充点
- `flatMap` 不是“更高级的 map”，它的核心价值在于“展开多层结构”。
- 这题很适合顺手提一句：`map` 解决转换，`flatMap` 解决嵌套结构拍平。

#### 面试回答模板
- `map` 和 `flatMap` 的区别在于：`map` 是把一个元素映射成一个新元素，适合普通转换；`flatMap` 是把一个元素映射成一个流或集合后，再把多层结构打平成一层，所以特别适合处理嵌套集合、字符串拆分、流中流这类场景。

#### 面试简答版
- `map`：转换。
- `flatMap`：转换 + 打平。

## 8. collect 和 reduce 有什么区别？

#### 先给结论
- `reduce` 更偏“把多个元素规约成一个值”。
- `collect` 更偏“把多个元素收集成某种结果容器或统计结构”。
- 面试里最简单的记法：
  - `reduce`：做和、积、最大值这类合并
  - `collect`：做 `List`、`Set`、`Map`、分组、拼接、统计

#### 核心解释
##### 1）`reduce`
- 更偏函数式的“归并”思想。
- 比如求和、求乘积、求最大值。

##### 2）`collect`
- 更偏工程化结果组织。
- 比如收集成列表、分组、转 Map、字符串拼接。

#### 简单代码示例
```java
import java.util.Arrays;
import java.util.List;
import java.util.stream.Collectors;

public class Demo {
    public static void main(String[] args) {
        List<Integer> list = Arrays.asList(1, 2, 3, 4);

        int sum = list.stream().reduce(0, Integer::sum);
        List<Integer> result = list.stream().map(x -> x * 10).collect(Collectors.toList());

        System.out.println(sum);
        System.out.println(result);
    }
}
```

#### 输出结果
```text
10
[10, 20, 30, 40]
```

#### 输出 / 现象解释
- `reduce` 把多个数字最终规约成了一个总和 `10`
- `collect` 把转换后的元素重新组织成了一个 `List`

#### 面试易错点 / 补充点
- 不要死记“都能收集结果”，而要抓本质：
  - `reduce` 更像“算出一个值”
  - `collect` 更像“装配结果”
- 分组统计题通常优先想到 `collect(Collectors.groupingBy(...))`

#### 面试回答模板
- `reduce` 和 `collect` 都属于 Stream 的终止操作，但侧重点不同。`reduce` 更适合把多个元素规约成一个值，比如求和、求最大值；`collect` 更适合把流中的元素收集成集合、Map，或者完成分组、拼接、统计等更复杂的结果组织。

#### 面试简答版
- `reduce`：规约成一个值。
- `collect`：收集成容器或统计结构。

## 9. Stream 会修改原集合吗？能重复使用吗？

#### 先给结论
- 一般不会修改原集合。
- 一般不能重复使用。
- 这是 Stream 两个非常高频的特性。

#### 核心解释
##### 1）不会主动修改数据源
- Stream 更像“在原数据上做计算”，而不是“直接改原数据”。
- 大多数操作会产生新结果，而不是修改原集合本身。

##### 2）不能重复使用
- 一个流一旦执行了终止操作，就相当于已经被消费掉了。
- 再次使用通常会抛 `IllegalStateException`。

#### 简单代码示例
```java
import java.util.Arrays;
import java.util.List;
import java.util.stream.Stream;

public class Demo {
    public static void main(String[] args) {
        List<Integer> list = Arrays.asList(1, 2, 3, 4);
        Stream<Integer> stream = list.stream().filter(x -> x > 2);

        System.out.println(stream.count());
        // System.out.println(stream.count()); // 运行时报错

        System.out.println(list);
    }
}
```

#### 输出结果
```text
2
[1, 2, 3, 4]
```

#### 输出 / 现象解释
- 第一次 `count()` 正常执行。
- 第二次如果再用同一个流，会报“stream has already been operated upon or closed”。
- 同时原始集合 `list` 没有变化。

#### 面试易错点 / 补充点
- “一般不修改原集合”不等于“绝对不会有副作用”，如果你在 `forEach` 或 `peek` 里手动改对象内部状态，那当然可能变。
- 真正稳的说法是：Stream 操作本身不会像 `add/remove` 那样去改集合结构，但你自己写的行为逻辑仍然可能带来副作用。

#### 面试回答模板
- Stream 一般不会修改原数据源，它更像是在数据源之上做一条计算流水线，最终产生新结果。另外，Stream 具有一次性消费特性，终止操作执行后流通常不能再复用，否则可能抛出 `IllegalStateException`。

#### 面试简答版
- 通常不改原集合。
- 一个流一般只能用一次。

## 10. parallelStream 怎么理解？什么时候用？

#### 先给结论
- `parallelStream()` 是并行流。
- 它的目标是利用多线程并行处理数据，提高某些场景下的吞吐量。
- 但它不是“用了就一定更快”，更不是“默认优于普通 stream()”。

#### 核心解释
##### 1）什么时候可能有收益
- 数据量比较大
- 每个元素的处理逻辑比较耗时
- 任务之间彼此独立
- 对顺序要求不高

##### 2）什么时候不建议乱用
- 数据量很小
- 任务本身很轻
- 需要严格顺序
- 存在共享可变状态
- 对线程池和上下文切换成本敏感

#### 简单代码示例
```java
import java.util.Arrays;
import java.util.List;

public class Demo {
    public static void main(String[] args) {
        List<Integer> list = Arrays.asList(1, 2, 3, 4, 5);

        list.parallelStream().forEach(System.out::println);
    }
}
```

#### 输出结果
```text
输出顺序可能不是 1 2 3 4 5
```

#### 输出 / 现象解释
- 并行流底层会把任务拆开并发执行。
- 所以 `forEach` 输出顺序可能和原集合顺序不一致。
- 如果你要保序，通常要考虑 `forEachOrdered()`。

#### 面试易错点 / 补充点
- 并行流不是万能优化按钮。
- 面试里最好主动补一句：并行流要避免共享可变状态，否则很容易出并发问题。
- 还可以补充一点：`findAny()` 在并行流里通常比 `findFirst()` 更容易发挥并行优势。

#### 面试回答模板
- `parallelStream()` 表示并行流，适合在数据量大、单个元素处理成本高、任务之间相互独立且对顺序要求不强的场景下使用。它并不保证一定更快，因为并行也会带来线程切换、拆分合并等额外开销，所以是否使用应结合具体场景和测试结果判断。

#### 面试简答版
- `parallelStream` = 并行流。
- 适合大数据量、重计算、无共享状态、顺序要求不高的场景。
- 不一定更快。
