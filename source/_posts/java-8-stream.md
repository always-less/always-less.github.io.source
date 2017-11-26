---
title: Java 8 Stream Api的一点研究
date: 2017-11-26 13:37:55
tags: Java
description: 可以说lambda和stream api彻底改变了Java程序员的代码风格及编程思想，它可以让你用声明性的方式处理数据集，用函数式的思维分析问题，并能够很轻易地并行处理数据。本篇将详细讲解stream api是如何实现这些强大的功能。
---

集合是Java中使用最多的API，几乎每个Java应用程序都会涉及到集合的处理。Java提供了丰富而强大的集合框架，但在Java8之前，对于一些集合聚合操作的处理上仍然只能靠程序员自己去处理集合的遍历、过滤、组装和增删改的各种细节。对于熟悉命令式编程的程序员来说会觉得编程本该如此，将实际问题用计算机熟悉的方式编程解决，而问题恰恰在此。当你从一行行的处理细节中试图还原代码的真实意图时就能体会到，命令式的风格是计算机所擅长的方式而非人类所易于接受的思维。

看下Stream Api是如何解决这个问题的。

# 告别for循环

举几个例子。

需求一：返回给定规则列表中规则优先级大于50的规则的名字列表。
实体类Rule
```java
@Data
public class Rule {
    private String name;    // 规则名称
    private int priority;   // 规则优先级
    private Action action;  // 规则决策结果

    public enum Action {
        DENY, PASS
    }
}
```
在Java8之前实现如下，
```java
public List<String> getRuleNames(List<Rule> rules) {
    List<String> ret = new ArrayList<>();
    for (Rule rule : rules) {
        if (rule.getPriority() > 50) {
            ret.add(rule.getName());
        }
    }
    return ret;
}
```
用stream api实现如下，
```java
public List<String> getRuleNamesByStream(List<Rule> rules) {
    return rules.stream()
            .filter(rule -> rule.getPriority() > 50)
            .map(Rule::getName)
            .collect(Collectors.toList())
}
```
虽然代码量差不多，但显然是两种不同的风格。下面的代码中，stream()函数用于将一个集合构造成一个流，然后filter()和map()函数分别定义了对于流里的每个元素的过滤和映射操作，collect()函数定义了流的终止操作，即将流里的每个元素收集到一个List中并返回。这里只直观感受两者区别，具体细节后面会展开分析。

如果你觉得也没看出stream api有多大优势，那么下面这两个例子肯定会改变你的想法。

需求二：对于给定的规则列表，跳过前10个，然后在剩下的规则中返回优先级大于50的前100个规则的名字列表。
在Java8之前实现如下，
```java
public List<String> getRuleNames(List<Rule> rules) {
    List<String> ret = new ArrayList<>();
    int count = 0;
    int skippedNum = 0;
    for (Rule rule : rules) {
        if (++skippedNum >= 10) {
            continue;
        }
        if (rule.getPriority() > 50) {
            if (++count > 100) {
                break;
            }
            ret.add(rule.getName());
        }
    }
    return ret;
}
```
可以看到虽然只是在第一个需求的基础上加了两个限制条件，但用命令式的方式要想bug-free的实现也还是要费一番工夫的。你可能需要仔细确认是要前缀++还是后缀++，该用>还是>=，应该在add前break还是应该在add后break。实际上这段代码不仅极为脆弱，且很难直观看出其意图。哪怕你知道需求，每次你看到这里的时候，可能都得再次确认是不是满足了需求。如果工程里全是这种代码，那维护的成本可想而知。

如果用stream api实现，则不能更简单，
```java
public List<String> getRuleNamesByStream(List<Rule> rules) {
    return rules.stream()
            .skip(10)
            .filter(rule -> rule.getPriority() > 50)
            .limit(100)
            .map(Rule::getName)
            .collect(Collectors.toList())
```
只需要“告诉”流需要先skip前10个规则，filter后在满足条件的规则中limit前100个，并把规则map成规则的名字collect到List中返回即可。几乎就是用代码把需求完整地复述了一遍，且stream api会帮你处理所有的底层细节。

需求三：在需求二的基础上，将返回的名字列表基于规则的Action分类返回。
在Java8之前实现如下，
```java
public Map<Rule.Action, List<String>> getRuleNames(List<Rule> rules) {
    Map<Rule.Action, List<String>> ret = new EnumMap<>(Rule.Action.class);
    int count = 0;
    int skippedNum = 0;
    for (Rule rule : rules) {
        if (++skippedNum >= 10) {
            continue;
        }
        if (count > 100) {
            break;
        }
        if (rule.getPriority() > 50) {
            if (ret.get(rule.getAction()) == null) {
                ret.put(rule.getAction(), new ArrayList<>());
            }
            ret.get(rule.getAction()).add(rule.getName());
            count++;
        }
    }
    return ret;
}
```
用stream api实现如下，
```java
import static java.util.stream.Collectors.*;
...

public Map<Rule.Action, List<String>> getRuleNamesByStream(List<Rule> rules) {
    return rules.stream()
            .skip(10)
            .filter(rule -> rule.getPriority() > 50)
            .limit(100)
            .collect(groupingBy(Rule::getAction, mapping(Rule::getName, toList())));
```

可以看到stream api可以用极为简单的方式实现类似Sql中group by的功能，而对于第一种方式应该也不用再吐槽了。

# 流

从上面的三组例子可以看到，stream api对于集合的处理都基于流（stream）这一抽象概念。
当把这一切都准备好让你可以开箱即用时，你可能大呼过瘾的同时会感觉对集合的处理本该如此。但能从零到一想到这种抽象，且能“完美”结合Java已有的集合框架以及新特性lambda表达式构建出这一整套体系，不得不说还是很强大的，这里面的技术思想也很值得一探。

## 流是什么

流是stream api对集合处理所定义的一个抽象概念，它允许你以声明式的方式处理数据集合。可以简单理解成遍历数据集的一个高级的内部迭代器，区别于传统Java集合中需要程序员外部控制遍历的iterator。流只能遍历一次，且可以透明的并行处理。

下图可助于理解stream的概念及处理过程。

![](/images/java-8-stream/stream.PNG)

几个关键问题，

- 上图中的处理过程中共遍历几遍集合？
> 一遍。具体的，是对每个元素先filter，如果符合条件再map成规则名，在判断如果当前数目小于3则加到待返回List中，如果等于3则加到List中返回，后面的元素不再遍历（短路）。

- filter、map、limit和collect有什么区别？
> 流的操作分为两种类型，filter、map、limit这种为intermediate操作，而collect为terminal操作。intermediate操作定义了流上每个元素要执行的具体操作，且都是惰性的（不会触发流的遍历），而terminal操作会触发流的遍历，并把符合条件的元素加入到自定义的集合中返回。

- 性能如何？
> 可以预想由于引入了新的数据结构的构造执行流程，性能上肯定不如同等逻辑的自定义for循环的代码。但还是那个老生常谈的问题，0.00001比0.0001快10倍，但真的matter吗？另外后面会讲并行流如何处理数据，这意味着我们可以非常轻松地在多核的机器上并行处理数据以提升性能。

## 流的使用

### 构造

构造流的几种常见方法如下
```java
// 1. 自定义
Stream stream = Stream.of("a", "b", "c");
IntStream intStream = IntStream.range(0, 3);    // 0, 1, 2
intStream = IntStream.rangeClosed(0, 3)         // 0, 1, 2, 3
intStream = IntStream.iterate(0, i -> i + 1);   // 递增的无穷大序列

// 2. 数组
String[] strArr = {"a", "b", "c"};
stream = Stream.of(strArr);
stream = Arrays.stream(strArr);

// 3. 集合
List<Rule> ruleList = new ArrayList<>();
stream = ruleList.stream();
```
需注意，对于基本类型，目前有三种对应的包装类型Stream：IntStream、LongStream、DoubleStream。这主要是考虑到数值类型集合运算的效率（不用boxing成对象类型和unboxing成基本类型），且数值类型流会有很多类似sum()、average()这种一般对象流中不会定义的终止操作。

### 操作
得到一个流后，便可以对流中的元素进行各种操作以返回我们想要的结果。常用的操作归类如下，
- Intermediate
> map、mapToInt、mapToLong、mapToDouble、flatMap、filter、distinc、sorted、skip、limit、peek、parallel、sequential

- Terminal
> forEach、collect、reduce、min、max、count、anyMatch、allMatch、noneMatch、findFirst、findAny

- Short-circuiting
> anyMatch、allMatch、noneMatch、findFirst、findAny、limit

具体使用方式可见jdk文档的注释。熟能生巧。

## 流的并行

在Java7之前，并行处理数据集合非常麻烦。第一，你得明确地把包含数据的数据结构分成若干子部分。第二，你要给每个子部分分配一个独立的线程。第三，你要在恰当的时候对它们进行同步来避免不希望出现的竞争条件，等待所有线程完成，最后把这些结果合并起来。Java7引入了Fork/Join框架，让这些操作更稳定且不易出错，但Fork/Join框架的使用仍然较为繁琐。现在，stream api也使用了Fork/Join框架，但封装了底层细节使你可以很轻易得实现数据集合的并行处理。当然要想正确使用，你仍需了解流内部是如何工作的。

### 使用
假设你需要计算一个巨型的int数组的和(假定不用考虑溢出的问题)，如下，
```java
public static int sum(int[] arr) {
    return Arrays.stream(arr)
            .reduce(0, Integer::sum);
}
```
这段代码等价于
```java
public static int sum(int[] arr) {
    int result = 0;
    for (int i = 0; i < arr.length; i++) {
        result += arr[i];
    }
    return result;
}
```
如果想要并行处理的话，只需要
```java
public static int sum(int[] arr) {
    return Arrays.stream(arr)
            .parallel()     // 将流转换为并行流
            .reduce(0, Integer::sum);
}
```
如此，stream api会帮你搞定所有的并行处理的细节，包括数据集合的划分、线程的创建、任务的分配、结果的合并。

> 如果想要使并行流变成顺序流只需调用sequential即可。两种调用不意味着流本身有任何实际的变化，其内部实现上只是设置了一个boolean标志，表示你所构造的流是并行流还是顺序流。就是说如下所示调用
> ```java
stream.parallel()
      .filter(...)
      .sequential()
      .map(...)
      .parallel()
      .reduce(...);
    ```
> 并不会并行执行filter、reduce操作而顺序执行map操作。最后一次parallel调用会使得整个流为并行流，且流上定义的所有操作都会并行执行

### 原理

#### Fork/Join框架

Fork/Join框架的目的是以递归的方式将可以并行的任务拆分成更小的任务，然后将每个子任务的结果合并起来生成整体结果。它是ExecutorService接口的一个实现，把子任务分配给线程池（ForkJoinPool）中的工作线程。

##### 定义任务
要把任务提交到ForkJoinPool，必须创建RecursiveTask<R>的一个子类，其中R是并行化任务（以及所有的子任务）产生的结果类型，或者如果任务不返回结果，则是RecursiveAction类型。要定义RecursiveTask，只需实现它唯一的抽象方法compute，
```java
protected abstract R compute();
```
这个方法同时定义了将任务拆分成子任务的逻辑，以及无法拆分或不方便拆分时，生成单个子任务结果的逻辑。方法实现的伪代码大致如下，
```java
if (任务足够小或不可分) {
    顺序计算该任务
} else {
    将任务分成两个子任务
    递归调用本方法，拆分每个子任务，等待所有子任务完成
    合并每个子任务的结果
}
```

一般来说没有确切的标准决定一个任务是否应该再拆分，但有几种试探方法可以参考，下面会提到。递归任务拆分过程如下所示，
![](/images/java-8-stream/fork-join-task.PNG)

用前面int数组求和的例子，定义ForkJoinSumCalculator任务如下，
```java
public class ForkJoinSumCalculator extends RecursiveTask<Integer> {
    private final int[] numbers;
    private final int start;
    private final int end;

    public static final int THRESHOLD = 10_000;

    public ForkJoinSumCalculator(int[] numbers) {
        this(numbers, 0, numbers.length);
    }

    private ForkJoinSumCalculator(int[] numbers, int start, int end) {
        this.numbers = numbers;
        this.start = start;
        this.end = end;
    }

    @Override
    protected int compute() {
        int length = end - start;
        if (length <= THRESHOLD) {
            return computeSequentially();
        }
        ForkJoinSumCalculator leftTask = 
                new ForkJoinSumCalculator(numbers, start, start + length / 2);
        leftTask.fork();    // 利用另一个ForkJoinPool线程异步执行新创建的子任务
        ForkJoinSumCalculator rightTask = 
                new ForkJoinSumCalculator(numbers, start + length / 2, end);
        int rightResult = rightTask.compute();  // 同步执行第二个子任务，有可能进一步递归划分
        int leftResult = leftTask.join();       // 读取第一个子任务结果，如果未完成就等待
        return leftResult + rightResult;    
    }

    private int computeSequentially() {
        int sum = 0;
        for (int i = start; i < end; i++) {
            sum += numbers[i];
        }
        return sum;
    }
}
```

##### 执行任务

如此再计算数组求和就比较简单了，
```java
public static int forkJoinSum(int[] numbers) {
    ForkJoinTask<Integer> task = new ForkJoinSumCalculator(numbers);
    return new ForkJoinPool().invoke(task);
}
```
实际应用中，使用多个ForkJoinPool是没有意义的，一般将其构造成单例重用。
本例中的计算过程如下，
![](/images/java-8-stream/fork-join-process.PNG)

##### 工作窃取

上例中在数组不多于10000个项目时就不再创建子任务了，这个选择是很随意的，但大多数情况下也很难找到一个好的启发式方法来确认它。如果有一个有1000万长度的数组，意味着ForkJoinSumCalculator会至少分出1000个子任务，对于多数计算机来说，似乎有点浪费资源，但分出大量的小任务一般来说都是一个好的选择。这是因为理想情况下，划分并行任务是应该让每个子任务都用完全相同的时间完成，让所有CPU内核都同样繁忙。但实际中，每个子任务所花的时间可能天差地别，要么因为划分策略效率低，或者其它不可预知的原因，如磁盘访问慢或者外部网络调用等。
Fork/Join框架的实现用work stealing（工作窃取）的技术来解决这个问题。在实际应用中，这意味着这些任务差不多被平均分配到ForkJoinPool的所有线程上，每个线程都为分配给它的任务保存一个双向链式队列，每完成一个任务就会从队列头取下一个任务开始执行。基于前面所述的原因，某个线程可能早早完成了分配给它的所有任务，也就是它的队列已经空了，而其他线程还很忙。这时这个线程会随机从一个别的线程的队列尾部“偷走”一个任务。这个过程一直持续下去，直到所有的任务都执行完毕。

##### 最佳实践
虽然Fork/Join框架还算简单易用，但它也很容易被误用。以下是几个有效使用它的最佳实践，
- 对一个任务调用Join方法会阻塞调用方，直到该任务作出结果。因此，有必要在两个子任务的计算都开始之后再调用join。
- 不应该在RecursiveTask内部使用ForkJoinPool的invoke方法。
- 对子任务调用fork方法可以把它排进ForkJoinPool，且一般对左右任务中一个任务fork，另一个复用当前线程直接compute，避免左右两边的子任务都调用fork而多分配一个任务。
- 多核处理器上使用Fork/Join框架并不一定比顺序计算快。要考虑问题的规模，分析一个任务是否可以分解成独立的子任务并进行合并，同时也要注意多线程编程的问题，如共享变量的访问等。
- 由于工作窃取机制的使用，控制任务分解的条件以能够分解出大量的小任务通常来说都是一个好的选择。

#### Spliterator

回到前面所述的并行流计算数组和的例子，其执行过程大致如下图所示，
![](/images/java-8-stream/parallel-stream.PNG)

可以看到整个过程就是利用Fork/Join框架不断切分子任务，交由ForkJoinPool里的线程处理，然后规约合并。但是我们的代码中并没有定义如何划分子任务的逻辑，这说明肯定有一种自动机制来帮我们做了流的拆分。这个新机制就是Spliterator。

Spliterator是Java8中加入的一个新接口，是为了可以并行遍历数据源中的元素而设计的。Spliterator接口定义如下，
```java
public interface Spliterator<T> {
    boolean tryAdvance(Consumer<? super T> action);
    Spliterator<T> trySplit();
    long estimateSize();
    int characteristics();
}
```
其中，T为Spliterator遍历的元素的类型。tryAdvance方法的行为类似于普通的Iterator，因为它会按顺序一个一个使用Spliterator中的元素，如果还有其他元素要遍历就返回true。trySplit是专为Spliterator接口设计，它可以把一些元素划分出去分给第二个Spliterator（由该方法返回），让它们两个并行处理。Spliterator还可以通过estimateSize()方法估计还剩下多少元素需要遍历，因为即使不那么确切，能快速算出来是一个值也有助于让拆分均匀一点。

将Stream拆分成多个部分的算法是一个递归过程，类似Fork/Join框架中RecursiveTask的拆分，Spliterator调用trySplit方法生成第二个Spliterator，然后对这两个Spliterator调用trySplit，如此直到它返回null，表明其所处理的数据结构不能再分割。这个拆分过程也受Spliterator本身的特性影响，而特性是通过characteristics方法声明的。Spliterator的特性如下表所示，

特性 | 含义 
--- | --- 
ORDERED | 元素由既定的顺序（例如List），因此Spliterator在遍历和划分时也会遵循这一顺序
DISTINCT | 对于任意一对遍历过的元素x和y，x.equals(y)返回false
SORTED | 遍历的元素按照一个预定义的顺序排序
SIZED | 该Spliterator由一个已知大小的源建立（例如Set），因此estimateSize()返回的是准确值
NONNULL | 保证遍历的元素不会为null
IMMUTABLE | Spliterator的数据源不能修改。这意味着在遍历时不能添加、删除或修改任何元素
CONCURRENT | 该Spliterator的数据源可以被其他线程同时修改而无需同步
SUBSIZED | 该Spliterator和所有从它拆分出来的Spliterator都是SIZED

默认情况下，Spliterator由框架提供，且流的数据源的结构和流上所定义的操作决定了Spliterator的特性，在使用并行流时需要额外注意。比如基于数组的ArrayList要比基于链表的LinkedList易于拆分处理，但如果使用并行流往ArrayList里添加数据的话就会出问题，因为ArrayList不是线程安全的。



> 并行流内部使用了默认的ForkJoinPool，默认的线程数为处理器（核心）的数量，这个值由Runtime.getRuntime().availableProcessors()得到，且可以通过系统属性 java.util.concurrent.ForkJoinPool.common.parallelism 来改变线程池大小，如下所示，
> ```java
System.setProperty("java.util.concurrent.ForkJoinPool.common.parallelism", "12");
    ```
> 需注意，这是一个全局变量，因此它将影响代码中所有的并行流。目前还无法专为某个并行流指定这个值，且无法为某个并行流指定特定的线程池。这意味着如果你代码中的某处并行流执行了某些比较耗时的操作，会影响其它地方并行流的性能，且不易发现。
> 一般而言，让ForkJoinPool等于处理器数量是个不错的默认值，除非有很好的理由，否则强烈建议不要修改。

### 限制

并不是所有的集合操作都适合并行流，且一般而言，想给出任何关于什么时候该用并行流的定量建议都是不可能也是毫无意义的。通常并行流的使用需要考虑以下几个方面，
- 如果不确定，测量。并行流并不一定总是比顺序流快，如果不确定并行流的引入是否会带来性能上的提升，建议用适当的基准测试来检查其性能。
- 留意装箱。自动装箱和拆箱会大大降低性能，Java8中有原始类型流来避免这种操作，但凡有可能都应该是使用这些流。
- 有些操作本身在并行流上的性能就比顺序流差。特别是limit和findFirst等依赖于元素顺序的操作，它们在并行流上执行的代价非常大。
- 要考虑才做流水线的总计算成本。设*N*是要处理的元素的个数，*Q*是一个元素通过流水线的大致处理成本，则*N* \* *Q*就是对成本的一个粗略的定性估计。*Q*值较高就意味着使用并行流的性能好的可能性比较大。
- 对于较小的数据量，选择并行流几乎永远都不是一个好的决定。
- 考虑背后的数据源的结构是否易于分解。例如基于数据的结构的拆分效率要远高于基于链表的结构，因为前者不用遍历就可以平均拆分。
- 考虑线程安全性。
- 流自身的特点，以及流水线中的中间操作修改流的方式，都有可能会改变分解过程的性能。例如，一个SIZED流可以分成大小相等的两部分，这样每部分都可以比较搞笑地并行处理，但filter操作可能丢弃的元素个数却无法预测，这导致流本身的大小未知。
- 考虑终端操作中合并步骤的代价是大是小（如Collector中的combiner方法）。如果这一步代价很大，那么组合每个子流的部分结果所付出的代价就可能会超过通过并行流得到的性能提升
- 还要注意，一个JVM进程中的所有并行流使用的是同一个共享的ForkJoinPool，如果一个并行流中的任务比较耗时则可能会间接地影响其它并行流的性能，而这往往很难察觉。

# 源码分析