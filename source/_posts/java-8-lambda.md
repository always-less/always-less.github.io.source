---
title: Java 8 Lambda表达式的一点研究
tags: Java
description: >-
  半年前在公司内部做过一次Java8 Stream Api的分享，在这里重新整理下，拆分为两篇。本篇主要讲解Java8
  lambda表达式底层原理，下篇着重分析Java8 Stream Api源码实现以及关于函数式编程的一点思考。
date: 2017-11-20 00:43:08
---


# Java 8 的变革
Java自面世以来已取得巨大的成功，但其繁复冗杂的语法也一直饱受诟病。我们叹息Java的不思进取却往往忽略了其所背负的历史包袱的沉重。作为一个历二十余年构建起的庞大生态的基石和根本，Java本就不可能像其他的一些JVM语言那样奔放和潇洒。然而即便如此，Java8所带来的新特性还是给予了开发者足够的惊喜。所有的变革中最受瞩目的毫无疑问是lambda表达式和stream api。

# Lambda 表达式
## 什么是lambda？
lambda表达式源于数学中的λ演算，在编程语言中可以简单的将其理解为匿名函数或闭包。很多语言（如Python、JavaScript）都支持lambda表达式，并允许将函数作为一个方法的参数。对于熟悉函数式编程的开发者来说，这一语法特性天经地义，但如果想在Java中实现类似效果，则要费一番功夫。
举例来说，我们有一个实体类Rule，
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
有一个处理方法是输出满足一定过滤条件的规则列表
```java
void processRules(List<Rule> rules, FilterStrategy filterStrategy) {
    for (Rule rule : rules) {
        if (filterStrategy.test(rule)) {
            System.out.println(rule);
        }
    }
}
```
其中FilterStrategy为一个过滤策略的抽象
```java
interface FilterStrategy {
    boolean test(Rule rule);
}
```
现在我们的客户端需要调用processRule()方法输出给定的规则列表中规则优先级大于50的规则，在Java8之前我们可以使用匿名类实现如下，
```java
void clientCallBeforeJava8(List<Rule> rules) {
    processRules(rules, new FilterStrategy() {
        @Override
        public boolean test(Rule rule) {
            return rule.getPriority() > 50;
        }
    });
}
```
可以看到我们写了很多定义一个过滤规则优先级大于50的FilterStrategy的代码，但这里面最核心要表达的逻辑其实只是`rule.getPriority() > 50`。再来看一下用Java8的lambda表达式如何实现这段逻辑，
```java
void clientCallInJava8(List<Rule> rules) {
    processRules(rules, rule -> rule.getPriority() > 50);
}
```
这里的`rule -> rule.getPriority() > 50`便是Java8的lambda表达式，由参数列表，->符号和函数体三部分组成。简洁且不失其语义，参数及返回值类型皆可由编译器推断。lambda表达式的更具体的使用方式及语法细节可参考[官方文档](https://docs.oracle.com/javase/tutorial/java/javaOO/lambdaexpressions.html)。
## 如何实现lambda
熟悉了lambda表达式的语法后，一个很自然的问题便是语法特性是如何实现的。分析上面给出的例子，我们大胆设想自己为Java8的设计者，且已敲定lambda表达式的使用方式即如上所示，那么基于Java平台的现状，可能的思路有以下几种。
### 编译期的解语法糖
最简单的方式便是将lambda表达式设计为上述匿名类方式的一个语法糖，即编译之后的class文件形式与匿名类实现方式编译后的结果一模一样。当然在将lambda表达式转换为字节码的过程中还会有method handles、dynamic proxies等其它方式，这几种方式各有其优缺点，不过其思路都是在编译期将lambda表达式转换成当前Java和JVM规范所支持的类型，具体细节在此不表。
如此对于Java8来说只需修改javac的编译器即可，class文件和JVM层面无需任何改动。

### 新的lambda类型
具体到上述例子，我们稍微修改下lambda表达式的定义方式，定义FilterStrategy为lambda类型，并在其元数据定义中声明lambda表达式的参数类型和返回数据类型，如下
```java
lambda FilterStrategy {
    Rule -> boolean
}
```
lambda类型的使用方式如下，
```java
void processRules(List<Rule> rules, FilterStrategy filterStrategy) {
    for (Rule rule : rules) {
        if (filterStrategy.lambda(rule)) {  // 用lambda()调用
            System.out.println(rule);
        }
    }
}
```
如此在使用上似乎也还蛮不错的。具体到lambda类型的实现上也会有两种思路，
- 作为interface类型的语法糖（一如当初被引入的enum类型的实现一样）
- class文件和JVM层面原生支持lambda类型

在此只讨论第二种的可行性，不过应该也没什么好讨论的。Java的类型系统自诞生之日起几乎从无变动，如若加入一个新的lambda类型，无论是语法层面还是底层JVM实现支持上都将会是一个大的改动，未必对得起所得收益。另一方面，lambda类型其本质即为一个函数，如此定义从使用角度看则像是函数脱离了类而独立存在，又与Java的纯面向对象的特点相悖。还有更重要的一点，如此设计很难与现有类库的兼容。比如Comparator接口其定义和使用上完美契合lambda表达式的使用场景，但是无法将其强行定义为此处所谓的lambda类型。

### Functional interface和invokedynamic指令
还是看下Java8的设计者之一Brian Goetz对于lambda表达式的设计文档中所提到的方案选择吧
> Translation strategy

> There are a number of ways we might represent a lambda expression in bytecode, such as inner classes, method handles, dynamic proxies, and others. Each of these approaches has pros and cons. In selecting a strategy, there are two competing goals: maximizing flexibility for future optimization by not committing to a specific strategy, vs providing stability in the classfile representation. We can achieve both of these goals by using the invokedynamic feature from JSR 292 to separate the binary representation of lambda creation in the bytecode from the mechanics of evaluating the lambda expression at runtime. Instead of generating bytecode to create the object that implements the lambda expression (such as calling a constructor for an inner class), we describe a recipe for constructing the lambda, and delegate the actual construction to the language runtime. That recipe is encoded in the static and dynamic argument lists of an invokedynamic instruction.

> The use of invokedynamic lets us defer the selection of a translation strategy until run time. The runtime implementation is free to select a strategy dynamically to evaluate the lambda expression. The runtime implementation choice is hidden behind a standardized (i.e., part of the platform specification) API for lambda construction, so that the static compiler can emit calls to this API, and JRE implementations can choose their preferred implementation strategy. The invokedynamic mechanics allow this to be done without the performance costs that this late binding approach might otherwise impose.

> When the compiler encounters a lambda expression, it first lowers (desugars) the lambda body into a method whose argument list and return type match that of the lambda expression, possibly with some additional arguments (for values captured from the lexical scope, if any.) At the point at which the lambda expression would be captured, it generates an invokedynamic call site, which, when invoked, returns an instance of the functional interface to which the lambda is being converted. This call site is called the lambda factory for a given lambda. The dynamic arguments to the lambda factory are the values captured from the lexical scope. The bootstrap method of the lambda factory is a standardized method in the Java language runtime library, called the lambda metafactory. The static bootstrap arguments capture information known about the lambda at compile time (the functional interface to which it will be converted, a method handle for the desugared lambda body, information about whether the SAM type is serializable, etc.)

> Method references are treated the same way as lambda expressions, except that most method references do not need to be desugared into a new method; we can simply load a constant method handle for the referenced method and pass that to the metafactory.

里面提到lambda新特性的添加需要考虑两个目标，一是转换策略要足够灵活以能够应对将来的优化，二是要维持现有class文件标准的稳定性。而invokedynamic指令可以同时满足这两个目标（invokedynamic指令自Java7引入，其时主要是为了更好的支持JVM上的动态类型语言的实现）。
具体的，Java8引入了函数式接口（只定义了一个抽象方法的接口）的概念来表示lambda的定义，上述的FilterStrategy即为一个函数式接口，同时新加了@FunctionalInterface的注解来标识一个函数式接口，如下所示
```java
@FunctionalInterface
interface FilterStrategy {
    boolean test(Rule rule);
}
```
如此如果该接口定义多个抽象方法在编译期就会报错。需要注意的是，接口的默认、静态方法以及声明了Object类中定义的public的方法的抽象函数（任何该接口的实现都一定会有Object的public方法的实现）都不被包括在内。
定义了函数式接口的方法参数都可以用lambda表达式代替，且编译后会生成一个invokedynamic的call site。用javap解析clientCallInJava8()方法所在class可得
```
void clientCallInJava8(java.util.List<cn.creditease.bdp.rule.Rule>);
    descriptor: (Ljava/util/List;)V
    flags:
    Code:
      stack=3, locals=2, args_size=2
         0: aload_0
         1: aload_1
         2: invokedynamic #9,  0              // InvokeDynamic #0:test:()Lcn/creditease/bdp/lambda/LambdaUsage$FilterStrategy;
         7: invokevirtual #8                  // Method processRules:(Ljava/util/List;Lcn/creditease/bdp/lambda/LambdaUsage$FilterStrategy;)V
        10: return
      LineNumberTable:
        line 31: 0
        line 33: 10
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0      11     0  this   Lcn/creditease/bdp/lambda/LambdaUsage;
            0      11     1 rules   Ljava/util/List;
      LocalVariableTypeTable:
        Start  Length  Slot  Name   Signature
            0      11     1 rules   Ljava/util/List<Lcn/creditease/bdp/rule/Rule;>;
    Signature: #40                          // (Ljava/util/List<Lcn/creditease/bdp/rule/Rule;>;)V
```
重点关注第8行`InvokeDynamic #0:test:()Lcn/creditease/bdp/lambda/LambdaUsage$FilterStrategy;`，其中#0代表BootstrapMethods属性表的第0项，即
```
BootstrapMethods:
  0: #64 invokestatic java/lang/invoke/LambdaMetafactory.metafactory:(Ljava/lang/invoke/MethodHandles$Lookup;Ljava/lang/String;Ljava/lang/invoke/MethodType;Ljava/lang/invoke/MethodType;Ljava/lang/invoke/MethodHandle;Ljava/lang/invoke/MethodType;)Ljava/lang/invoke/CallSite;
    Method arguments:
      #65 (Lcn/creditease/bdp/rule/Rule;)Z
      #66 invokestatic cn/creditease/bdp/lambda/LambdaUsage.lambda$clientCallInJava8$0:(Lcn/creditease/bdp/rule/Rule;)Z
      #65 (Lcn/creditease/bdp/rule/Rule;)Z
```
也就是说，当运行时执行到lambda表达式时，会调用invokedynamic指令，实际上执行的是LambdaMetafactory.metafactory()方法。根据jdk的注释可知，metafactory()方法的前三个参数会在运行时根据访问类和运行期常量池动态传入，而后三个参数则由bootstrao静态参数列表传入，即#65、#66、#65。#66是一个MethodHandle（方法句柄），代表了真正执行的方法定义，方法名为lambda$clientCallInJava8$0，且与方法clientCallInJava8位于同一class（LambdaUsage）。
Debug LambdaMetafactory.metafactory()源码可以发现，方法内部实际上还是构造了一个实现了@FunctionalInterface的匿名类，并返回一个CallSite链接到该对象（lambda object）。

这种方式和前面提过的在编译时直接desugar为匿名内部类的方式相比，除了可以显著减少编译后的字节码文件数量及大小，更为重要的区别是，它将lambda表达式的具体翻译策略的实现推迟到了运行时，并可以由类库的开发者自己定义。事实上从长远来说，Oracle是希望将lambda的翻译策略改为基于MethodHandle的MethodHandleProxy的方式，由于invokedynamic的运用，可以预想到将来的这项优化可以很轻松的实现（只需修改jdk的metafactory实现即可）。

更多关于lambda表达式的编译细节可参考[Translation of Lambda Expressions](http://cr.openjdk.java.net/~briangoetz/lambda/lambda-translation.html)。

# Stream Api
有了语言层面上的lambda表达式的原生支持，我们就可以在Java中尝试所谓的函数式编程。而Java8提供的Stream Api便是对在Java中进行函数式编程的一次精彩演绎。（详见下篇）