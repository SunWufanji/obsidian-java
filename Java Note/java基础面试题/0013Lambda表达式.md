# Lambda表达式

## 1. 什么是 Lambda 表达式？为什么要用它？

#### 先给结论
- Lambda 表达式是 Java 8 引入的一种“把行为当参数传递”的写法。
- 你可以先把它理解成：用更短的代码，去实现“只有一个抽象方法”的接口。
- 它最直接的价值有 3 个：
  - 简化匿名内部类写法
  - 让集合、回调、排序、过滤代码更清晰
  - 为 Stream 函数式编程提供基础

#### 核心解释
- 在 Java 8 之前，很多“传一段逻辑进去”的场景，通常都要写匿名内部类。
- 但如果接口里其实只有一个核心方法，这种写法会显得很啰嗦。
- Lambda 的本质就是：把“我要做什么”这段逻辑，直接写出来，而不是先写一个类再实现它。
- 你可以把它理解成：
  - 以前是“先造一个对象，对象里再写方法”
  - 现在是“直接把这个方法逻辑当参数传进去”

#### 简单代码示例
```java
public class Demo {
    public static void main(String[] args) {
        new Thread(() -> System.out.println("hello lambda")).start();
    }
}
```

#### 输出结果
```text
hello lambda
```

#### 输出 / 现象解释
- `Thread` 构造器需要的是一个 `Runnable`。
- `Runnable` 只有一个抽象方法 `run()`。
- 所以这里可以直接用 `() -> System.out.println("hello lambda")` 来表示“run 方法该做什么”。

#### 面试易错点 / 补充点
- 不要把 Lambda 只理解成“语法更短”，它更重要的意义是“让行为可以更自然地传递”。
- Lambda 不是随便哪里都能写，它必须有“目标类型”，通常是函数式接口。

#### 面试回答模板
- Lambda 表达式是 Java 8 引入的函数式写法，本质上是把行为当作参数传递。它通常用于替代只有一个抽象方法的接口实现，比如匿名内部类，从而让回调、排序、过滤、遍历等代码更简洁，也为 Stream API 提供了基础。

#### 面试简答版
- Lambda = 更简洁地传递一段行为逻辑。
- 常用于替代函数式接口的匿名内部类写法。

## 2. Lambda 表达式的使用前提是什么？

#### 先给结论
- Lambda 的使用前提是：目标类型必须是函数式接口。
- 函数式接口指的是：接口中只有一个抽象方法。
- 这个接口里可以有多个 `default` 方法和 `static` 方法，但抽象方法只能有一个。

#### 核心解释
- Lambda 不是凭空存在的，它一定要“落到某个类型上”。
- 编译器必须知道：你这段 Lambda 最终是要实现哪个方法。
- 如果一个接口里有多个抽象方法，编译器就不知道你到底在实现哪一个，所以不能用 Lambda。

#### 简单代码示例
```java
@FunctionalInterface
interface MyPrinter {
    void print(String s);
}

public class Demo {
    public static void main(String[] args) {
        MyPrinter p = str -> System.out.println(str);
        p.print("java");
    }
}
```

#### 输出结果
```text
java
```

#### 输出 / 现象解释
- `MyPrinter` 只有一个抽象方法 `print`，所以它是函数式接口。
- `str -> System.out.println(str)` 本质上就是这个 `print(String s)` 方法的实现。

#### 面试易错点 / 补充点
- `@FunctionalInterface` 不是必须写，但非常推荐写。
- 它的好处是：如果你不小心又加了第二个抽象方法，编译器会直接报错。
- `Object` 里的方法不算新增抽象方法，比如 `toString()`、`equals()` 不影响函数式接口判断。

#### 面试回答模板
- Lambda 表达式必须依附于函数式接口使用。函数式接口指的是只包含一个抽象方法的接口，它可以额外包含默认方法和静态方法。编译器需要通过这个唯一抽象方法来确定 Lambda 到底在实现什么行为。

#### 面试简答版
- 前提：目标类型必须是函数式接口。
- 函数式接口 = 只有一个抽象方法的接口。

## 3. Lambda 表达式的语法怎么写？有哪些常见简写规则？

#### 先给结论
- Lambda 最核心的语法只有一个：
```java
(参数列表) -> { 方法体 }
```
- 但实际使用时，经常会有很多简写。

#### 核心解释
##### 1）无参写法
```java
() -> System.out.println("hello")
```

##### 2）一个参数
```java
x -> System.out.println(x)
```
- 只有一个参数时，参数外面的括号可以省略。

##### 3）多个参数
```java
(a, b) -> a + b
```

##### 4）单行表达式
```java
(a, b) -> a + b
```
- 如果方法体只有一行表达式，可以省略 `{}` 和 `return`。

##### 5）多行代码块
```java
(a, b) -> {
    int sum = a + b;
    return sum;
}
```
- 如果是代码块，并且有返回值，就要显式写 `return`。

##### 6）参数类型推断
```java
(a, b) -> a + b
```
- 很多时候参数类型可以省略，编译器会根据上下文自动推断。

#### 简单代码示例
```java
interface Calc {
    int add(int a, int b);
}

public class Demo {
    public static void main(String[] args) {
        Calc c = (a, b) -> a + b;
        System.out.println(c.add(3, 5));
    }
}
```

#### 输出结果
```text
8
```

#### 输出 / 现象解释
- `(a, b) -> a + b` 对应的就是 `add(int a, int b)` 的实现。
- 因为 `Calc` 的方法签名已经告诉编译器参数和返回值类型，所以这里可以省略参数类型。

#### 面试易错点 / 补充点
- 只有一个参数时，括号可以省；多个参数时不能省。
- 单行表达式可以省略 `return`；多行代码块如果要返回值，就必须写 `return`。
- 参数类型要么都写，要么都不写，不能一半写一半不写。

#### 面试回答模板
- Lambda 的基本语法是 `(参数列表) -> { 方法体 }`。如果只有一个参数，可以省略参数括号；如果方法体只有一行表达式，可以省略大括号和 `return`；参数类型通常也可以由编译器根据目标类型自动推断出来。

#### 面试简答版
- 基本语法：`(参数) -> 表达式` 或 `(参数) -> { 代码块 }`
- 常见简写：省参数类型、省单参数括号、省单行 `return`

## 4. Java 中常见的函数式接口有哪些？怎么用？

#### 先给结论
- 面试里最常考的 4 个函数式接口是：
  - `Predicate<T>`：判断，返回 `boolean`
  - `Consumer<T>`：消费，接收参数，无返回值
  - `Function<T, R>`：转换，`T -> R`
  - `Supplier<T>`：提供，没参数，返回一个值

#### 核心解释
##### 1）`Predicate<T>`
- 用来做条件判断。
- 典型方法：`boolean test(T t)`

##### 2）`Consumer<T>`
- 用来消费一个对象。
- 典型方法：`void accept(T t)`

##### 3）`Function<T, R>`
- 用来做类型映射或值转换。
- 典型方法：`R apply(T t)`

##### 4）`Supplier<T>`
- 用来“提供一个值”。
- 典型方法：`T get()`

#### 简单代码示例
```java
import java.util.function.Consumer;
import java.util.function.Function;
import java.util.function.Predicate;
import java.util.function.Supplier;

public class Demo {
    public static void main(String[] args) {
        Predicate<Integer> p = x -> x > 10;
        Consumer<String> c = s -> System.out.println(s);
        Function<String, Integer> f = s -> s.length();
        Supplier<String> s = () -> "hello";

        System.out.println(p.test(12));
        c.accept("java");
        System.out.println(f.apply("lambda"));
        System.out.println(s.get());
    }
}
```

#### 输出结果
```text
true
java
6
hello
```

#### 输出 / 现象解释
- `Predicate` 做条件判断，所以 `12 > 10` 输出 `true`
- `Consumer` 负责消费字符串，所以直接打印 `java`
- `Function` 把 `"lambda"` 映射成长度 `6`
- `Supplier` 不接收参数，直接返回 `"hello"`

#### 面试易错点 / 补充点
- 不要只死背接口名，最好连“输入输出关系”一起记。
- 很多 Stream API 背后接收的就是这些函数式接口。
- 此外，`Runnable`、`Comparator`、`Callable` 也都可以作为 Lambda 的目标类型。

#### 面试回答模板
- Java 8 在 `java.util.function` 包里提供了很多标准函数式接口，其中最常用的是 `Predicate`、`Consumer`、`Function` 和 `Supplier`。它们分别对应判断、消费、转换和提供值的场景，也是 Stream API 和集合增强方法中最常出现的 Lambda 目标类型。

#### 面试简答版
- `Predicate`：判断
- `Consumer`：消费
- `Function`：转换
- `Supplier`：提供

## 5. Lambda 和匿名内部类有什么区别？

#### 先给结论
- Lambda 可以在很多场景下替代匿名内部类，但它不等于匿名内部类。
- 两者最重要的区别有 4 个：
  - Lambda 只能用于函数式接口
  - Lambda 写法更简洁
  - Lambda 的 `this` 指向外部对象，不是“新建出来的匿名类对象”
  - JVM 对 Lambda 的实现也不是简单生成一个匿名内部类 class 文件

#### 核心解释
##### 1）使用范围不同
- 匿名内部类可以实现普通接口，也可以继承抽象类。
- Lambda 只能实现函数式接口。

##### 2）语法层面不同
- 匿名内部类要写接口名、方法名、`@Override`。
- Lambda 直接写“参数 -> 逻辑”。

##### 3）`this` 含义不同
- 在匿名内部类里，`this` 指的是匿名内部类对象本身。
- 在 Lambda 里，`this` 指向外层对象。

##### 4）底层实现不同
- 第一篇文章特别强调了这一点：Lambda 不是简单把匿名内部类缩写一下。
- 匿名内部类通常会生成额外的 class 文件。
- Lambda 在 JVM 层面通常通过 `invokedynamic` 指令来支持。

#### 简单代码示例
```java
public class Demo {
    private String name = "Outer";

    public void test() {
        Runnable r = () -> System.out.println(this.name);
        r.run();
    }

    public static void main(String[] args) {
        new Demo().test();
    }
}
```

#### 输出结果
```text
Outer
```

#### 输出 / 现象解释
- 这里 Lambda 里的 `this.name` 访问的是外部 `Demo` 对象的字段。
- 这说明 Lambda 没有像匿名内部类那样引入一个新的 `this` 语义。

#### 面试易错点 / 补充点
- “Lambda 是匿名内部类的语法糖”这个说法，面试里说一半对、一半不够严谨。
- 更准确的说法是：
  - 从写法上看，它确实常用来简化匿名内部类
  - 但从 JVM 实现上看，它并不是简单编译成匿名内部类
- 可以顺手关联：[[0011语法糖#1. 什么是 Java 语法糖？当前这几章里哪些内容明确涉及语法糖？|0011 语法糖]]

#### 面试回答模板
- Lambda 经常用于简化匿名内部类，但两者不完全等价。Lambda 只能用于函数式接口，语法更简洁；同时它的 `this` 指向外层对象，而匿名内部类的 `this` 指向匿名类实例。从 JVM 实现角度看，Lambda 通常基于 `invokedynamic`，也不是简单生成一个匿名内部类文件。

#### 面试简答版
- 能替代部分匿名内部类，但不等于匿名内部类。
- 关键区别：函数式接口限制、`this` 语义不同、底层实现不同。

## 6. 为什么 Lambda 里引用的局部变量必须是 final 或 effectively final？

#### 先给结论
- 因为 Lambda 会捕获外部局部变量。
- 为了保证语义清晰、线程安全和实现简单，Java 要求这些被捕获的局部变量不能再被修改。
- 所以它不一定要显式写 `final`，但必须是“事实上没有再改过”的，也就是 effectively final。

#### 核心解释
- Lambda 可以访问外部局部变量，这叫变量捕获。
- 但局部变量本来是方法栈帧里的数据，生命周期和对象字段不一样。
- Lambda 在后续执行时，如果外部局部变量还能随意变，就会让语义变得很混乱。
- 所以 Java 干脆规定：
  - 你可以读
  - 但不能改

#### 简单代码示例
```java
import java.util.Arrays;
import java.util.List;

public class Demo {
    public static void main(String[] args) {
        List<Integer> list = Arrays.asList(1, 2, 3);
        int factor = 10;

        list.forEach(x -> System.out.println(x * factor));
        // factor++; // 编译报错：不是 effectively final
    }
}
```

#### 输出结果
```text
10
20
30
```

#### 输出 / 现象解释
- `factor` 虽然没写 `final`，但它后面没有再被修改，所以属于 effectively final。
- 因此 Lambda 可以安全读取它。
- 如果你把 `factor++` 放开，编译器就会报错。

#### 面试易错点 / 补充点
- 限制的是“局部变量”，不是成员变量。
- 成员变量、静态变量在 Lambda 里可以正常读写。
- 很多人会把“能访问外部变量”直接理解成“外部变量都能改”，这是错的。

#### 面试回答模板
- Lambda 可以捕获外部作用域中的局部变量，但这些局部变量必须是 `final` 或 effectively final。原因是局部变量存活在方法栈帧中，如果后续还能在 Lambda 内外随意修改，会让变量捕获的语义变得不清晰。Java 因此要求这类变量只读不改。

#### 面试简答版
- 因为 Lambda 会捕获局部变量。
- 为了保证语义稳定，被捕获的局部变量必须是 final 或 effectively final。

## 7. 什么是方法引用？它和 Lambda 的关系是什么？

#### 先给结论
- 方法引用可以看成 Lambda 的进一步简化。
- 当前提满足时，可以用“已有方法”直接替代一段很简单的 Lambda。
- 常见形式有 4 种：
  - `对象::实例方法`
  - `类名::静态方法`
  - `类名::实例方法`
  - `类名::new`

#### 核心解释
- 如果你的 Lambda 只是“把参数原封不动传给另一个方法”，那通常可以改成方法引用。
- 例如：
```java
list.forEach(x -> System.out.println(x));
```
- 可以简化为：
```java
list.forEach(System.out::println);
```

#### 简单代码示例
```java
import java.util.Arrays;
import java.util.List;

public class Demo {
    public static void main(String[] args) {
        List<String> list = Arrays.asList("a", "b", "c");
        list.forEach(System.out::println);
    }
}
```

#### 输出结果
```text
a
b
c
```

#### 输出 / 现象解释
- `System.out::println` 本质上就是把“调用 `println` 打印每个元素”这段逻辑直接拿来用。
- 它比 `x -> System.out.println(x)` 更短，也更直观。

#### 面试易错点 / 补充点
- 不是所有 Lambda 都能替换成方法引用。
- 只有当 Lambda 本质上只是“转调一个已有方法”时，才适合替换。
- 方法引用本质上依然依赖函数式接口，并不是独立于 Lambda 的另一套机制。

#### 面试回答模板
- 方法引用可以理解为 Lambda 的一种简化写法。当 Lambda 只是把参数传给现有方法执行时，可以用 `::` 语法直接引用该方法。常见形式包括对象实例方法引用、类静态方法引用、类实例方法引用以及构造器引用。

#### 面试简答版
- 方法引用 = 更简洁的 Lambda。
- 常见形式：`对象::方法`、`类::静态方法`、`类::实例方法`、`类::new`

## 8. Lambda 在实际开发中最常见的使用场景有哪些？

#### 先给结论
- 面试和开发里最常见的 4 个场景：
  - 遍历集合
  - 排序
  - 条件过滤
  - Stream 链式处理

#### 简单代码示例
```java
import java.util.Arrays;
import java.util.List;
import java.util.stream.Collectors;

public class Demo {
    public static void main(String[] args) {
        List<String> list = Arrays.asList("java", "go", "python", "js");

        list.forEach(System.out::println);

        list.sort((a, b) -> a.length() - b.length());
        System.out.println(list);

        List<String> result = list.stream()
                .filter(s -> s.length() > 2)
                .map(String::toUpperCase)
                .collect(Collectors.toList());

        System.out.println(result);
    }
}
```

#### 输出结果
```text
java
go
python
js
[go, js, java, python]
[JAVA, PYTHON]
```

#### 输出 / 现象解释
- `forEach(System.out::println)`：遍历集合
- `sort((a, b) -> a.length() - b.length())`：自定义排序
- `filter(s -> s.length() > 2)`：条件过滤
- `map(String::toUpperCase)`：元素转换

#### 面试易错点 / 补充点
- Lambda 本身不是重点，重点是它和集合、Stream、函数式接口一起形成的开发风格。
- 所以面试官问 Lambda，后面很可能顺着追：
  - `forEach`
  - `sort`
  - `filter`
  - `map`
  - `Stream`

#### 面试回答模板
- Lambda 在实际开发中最常见的场景包括集合遍历、自定义排序、条件过滤以及 Stream 链式处理。比如 `forEach`、`sort`、`filter`、`map` 这些 API 都大量依赖函数式接口，因此和 Lambda 配合使用非常自然。

#### 面试简答版
- 常见场景：遍历、排序、过滤、Stream 处理。

## 9. 面试里怎么回答“Lambda 表达式的本质”？

#### 先给结论
- 从使用层面看，Lambda 就是函数式接口的简洁实现写法。
- 从语法层面看，它确实让匿名内部类场景更简洁。
- 但从 JVM 实现层面看，它不是简单替换成匿名内部类，而是和 `invokedynamic` 相关。

#### 核心解释
- 你可以分三层理解：
  - 第一层：会写，会用
  - 第二层：知道它依赖函数式接口
  - 第三层：知道它和匿名内部类在 JVM 层面不完全一样
- 对面试来说，第三层就是加分点。

#### 面试易错点 / 补充点
- 如果只是初中级面试，说到“简化匿名内部类 + 配合函数式接口 + 常见在 Stream 里用”通常就够了。
- 如果面试官继续深挖，再补：
  - Lambda 体在编译后通常会被处理成私有方法
  - 调用和 `invokedynamic` 有关

#### 面试回答模板
- Lambda 表达式本质上是 Java 8 提供的一种函数式编程写法，用来更简洁地实现函数式接口。它在使用层面经常替代匿名内部类，但 JVM 层面并不是简单生成匿名内部类，而是借助 `invokedynamic` 等机制来完成调用，因此它既有语法简化的一面，也有独立的底层实现方式。

#### 面试简答版
- 表面：更简洁地实现函数式接口。
- 深层：不只是匿名内部类简写，底层和 `invokedynamic` 相关。
