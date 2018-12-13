# 第二单元：架构

## Logback 架构

Logback 的基础架构非常通用，因此适用于很多不同的场景。下面将分为三个模块来说明，`logback-core`，`logback-classic`,`logback-access`。`core` 模块是另外两个模块的底层依赖。`classic` 模块扩展了 `core` 模块。`classic` 模块与 `log4j` 相比较有非常显著的改进。`logback-classic` 原生实现了 SLF4J API，因此可以方便的在 logback 和其他日志系统间进行切换，比如 log4j 或 在 JDK1.4 开始引入的 `java.util.logging`(JUL)。第三个模块 access 集成了 Servlet 容器，以此来提供 HTTP 访问日志的功能。access 模块会通过一个独立的文档进行说明。

在本章节的剩下部分，我们会用 logback 来指代 logback-classic 模块。


### Logger，Appenders 与 Layouts

Logback 的实现基于三个核心的类：Logger，Appender 和 Layout。这三个组件共同工作是开发者能够根据消息类型和级别进行日志输出，并在运行时控制消息的格式化与报告。

`Logger` 类是 logback-classic 模块的一部分。另一方面，Appender 和 Layout 接口是 logback-core 的一部分。作为一个通用的模块，logback-core 没有日志记录器（logger）的概念。


#### Logger 上下文

基于 System.out.print 的 日志输出 API 最重要的优势就是允许用户自由的打印或是关闭日志声明的能力。这个功能的前提是假设日志空间（所有可能的日志输出声明的空间）以开发者选择的标准进行了分类。在 logback-classic 中，这种分类方式是日志记录器的固有组成部分。每个单独的日志记录器都会与一个 LoggerContext 进行关联，LoggerContext 负责生成大量的日志记录器，并将这些日志记录器以树形结构进行组织。

日志记录器都是命名实体。他们的名字大小写敏感，同时遵循集成命名规则：

> #### 命名体系
>
>一个日志记录器的名字附加上 `.` 组成的字符串如果是其他日志记录器名字的前缀，就认为这个日志记录器是这些日志记录器的祖先。如果一个日志记录器与后代日志记录器之间没有其他日志记录器，就成这个日志记录器是后代日志记录器的父日志记录器。
>
例如，命名为 “com.foo” 的日志记录器是 “com.foo.Bar” 的日志记录器的父日志记录器。类似的，“java” 是 “java.util” 的父日志记录器，并且是 “java.util.Vector” 的祖先。这种命名模式对大多数开发者来说应该都很熟悉。

根日志记录器（root logger）是日志记录器体系的顶级节点。根日志记录器是其他所有日志记录器体系的开始。与每个日志器一样，根日志记录器也可以通过名字来调用，如下：

```java
Logger rootLogger = LoggerFactory.getLogger(org.slf4j.Logger.ROOT_LOGGER_NAME);
```

其他所有日志记录器也可以通过调用 `org.slf4j.LoggerFactory` 类的 `getLogger` 方法来获取。该方法以目标日志记录的名字作为参数。下面列出了 Logger 接口的一些基础方法：

```java
package org.slf4j;

public interface Logger {
    public void trace(String message);
    public void debug(String message);
    public void info(String message);
    public void warn(String message);
    public void error(String message);
    
}
```


#### 有效级别又称级别继承

日志记录器可以分配级别。级别集合（TRACE，DEBUG，INFO，WARN，ERROR）在 `ch.qos.logback.classic.Level` 类中定义。在 logback 中，`Level` 是 final 类，无法被扩展子类，相比之前的 Marker 对象更加灵活。

如果一个日志记录器没有分配级别，会继承最近的已分配级别的祖先的级别。更正式的表述：

>一个给定的日志记录器 L 的有效级别，是在以 L 自身并向上直至根日志记录器的继承体系中第一个非 null 的级别。

为了确保所有日志记录器最终都能够继承一个级别，根日志记录器会总是被分配一个级别，默认会分配 DEBUG 级别。

下面四个示例展示了其分配的不同级别与最终根据继承规则生效的有效级别。

*示例 2.1*

|日志记录器名字|分配级别|有效级别|
|:------|:-----|:----|
|root|DEBUG|DEBUG|
|X|none|DEBUG|
|X.Y|none|DEBUG|
|X.Y.Z|none|DEBUG|

在示例 2.1 中，只有根日志记录器被分配了一个级别。日志级别 DEBUG 被其他日志记录器继承。

*示例 2.2*

|日志记录器名字|分配级别|有效级别|
|:------|:-----|:----|
|root|ERROR|ERROR|
|X|INFO|INFO|
|X.Y|DEBUG|DEBUG|
|X.Y.Z|WARN|WARN|

在示例 2.2 中，所有日志记录器都被分配了一个级别。级别继承作机制不起作用。

*示例 2.3*

|日志记录器名字|分配级别|有效级别|
|:------|:-----|:----|
|root|DEBUG|DEBUG|
|X|INFO|INFO|
|X.Y|none|INFO|
|X.Y.Z|ERROR|ERROR|

在示例 2.3 中，根日志记录器，X 和 X.Y.Z 日志记录器被分别分配了级别 DEBUG，INFO，ERROR。X.Y 日志记录器继承了它的父日志记录器 X 的级别。

*示例 2.4*

|日志记录器名字|分配级别|有效级别|
|:------|:-----|:----|
|root|DEBUG|DEBUG|
|X|INFO|INFO|
|X.Y|none|INFO|
|X.Y.Z|none|INFO|

在示例 2.4 中，根日志记录器与 X 日志记录器分别被分配了级别 DEBUG，INFO。日志记录器 X.Y 和 X.Y.Z 继承了他们的最近有分配级别的父日志记录器 X。


#### 打印方法和基本的命中规则

日志打印方法在定义中确定了一个日志输出请求的日志级别。例如，如果 L 是一个日志记录器实例，则 L.info("..“) 就是一个日志级别为 INFO 的日志输出声明。

如果一个日志输出请求的级别等于或高于日志记录器的有效级别，就说该日志输出请求是激活的。否则，该日志输出请求是无效的。如之前的描述，一个没有分配级别的日志记录器会继承最近祖先的日志级别。这条规则在下面进行了总结。


>#### 基本命中规则
>
>如果一个级别为 *p* 日志请求非分发给了一个有效级别为 *q* 的日志记录器，则当 *p >= q* 时，日志请求生效。

这条规则是 logback 的核心。日志级别的顺序定义为：*TRACE > DEBUG > INFO > WARN > ERROR*。

为了更形象的说明命中规则的工作，在下面的表格中，纵向的表头是指定给日志记录器 p 的有效级别，表格中行与列交叉的每个格子显示了基本命中规则的结果。

![](https://raw.githubusercontent.com/rootedInSoil/logback_manual/master/img-2.1.png)


下面是一个基本命中规则的例子

```java
import ch.qos.logback.classic.Level;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
...

// 获取一个命名为 “com.foo” 的日志记录器实例。另外再假设该日志记录器的类型是 `ch.qos.logback.classic.Logger`，
//因此我们可以设置它的级别
ch.qos.logback.classic.Logger logger = (ch.qos.logback.classic.Logger)LoggerFactory.getLogger("com.foo");

// 设置级别为 INFO。setLevel（）方法要求是 logback 的日志记录器
logger.setLevel(Level.INFO);

Logger barLogger = LoggerFactory.getLogger("com.foo.Bar");

// 请求有效，因为 WARN >= INFO
logger.warn("Low fuel level.");

// 请求无效，因为 DEBUG < INFO
logger.debug("Starting search for nearest gas station.");

// The logger instance barlogger, named "com.foo.Bar", 
// will inherit its level from the logger named 
// "com.foo" Thus, the following request is enabled 
// because INFO >= INFO. 
barLogger.info("Located nearest gas station.");

// This request is disabled, because DEBUG < INFO. 
barLogger.debug("Exiting gas station search");
```
           
           
#### 调用日志记录器

用同一个名字参数调用 `LoggerFactory.getLogger` 返回的是同一个日志记录器对象的引用。

例如，
```java
Logger x = LoggerFactory.getLogger("wombat");
Logger y = LoggerFactory.getLogger("wombat");
```
`x` 和 `y` 引用的是统一个日志记录器对象。

因此，可以配置一个日志记录器然后再代码中任意位置获取相同的示例，而不过通过引用进行传递。与自然生物关系不一样，logback 里的日志记录器可以按任意顺序创建与发布。特别的，一个“父”日志记录器会寻找并关联它的后代，即便它在后代日志记录器创建后才实例化。

logback 环境在应用初始化时进行配置。推荐通过读取配置文件进行配置。这种方式很快会进行讨论。

Logback 使得通过组件进行日志记录器命名很容易。可以使用日志记录器所在类的完全类名进行命名。这是一种有效直接的日志记录器命名方式。在日志输出中包含日志记录器的名字，这种命名方法可以很容易定位到日志消息源。然而这只是一种相对通用的可能策略。Logback 并没有限制日志记录器集合的可能性。作为一个开发者，可以自由的按照意愿命名一个日志记录器。

尽管如此，在类被加载后在命名日志记录器是目前已知的一种最佳策略。

#### Appenders 与 Layouts

根据日志记录器来选择开启或关闭日志输出请求的能力只是 logback 的一部分。Logback 允许日志输出请求打印到多个输出目标。在 logback 的定义里，一个输出目标被称为 appender。目前，输出目标可以是终端，文件，远程套接字服务，Mysql，PostgreSQL，Oracle 或其他数据库，JMS 和 远程 Unix Syslog 进程。

可以将多个 appender 关联到一个日志记录器。

`addAppender` 方法可以添加一个 appender 到给定的日志记录器。所有开启的日志请求会被同时发送到日志记录器的所有 appender 与日志记录器体系中更高的 appender。换句话说，appender 会从日志记录器的继承体系中继承叠加。例如，如果一个终端 appender 被添加到了根日志记录器，那所有开启的日志请求至少有一个打印。如果日志记录器 *L* 另外添加了一个文件 appender，那么日志记录器 L 与 L 的子日志记录器的所有开启日志都会同时输出都文件和终端。通过设置日志记录器 additivity 属性标记值为 false 来禁用 appender 的继承叠加特性。

这个治理 appender 叠加属性的规则下面进行了总结。



>#### Appender Additivity
>
>日志记录器 L 的一个日志声明会被发送到 L 自己的所有 appender 和它的祖先。这就是 “appender additivity” 属于的定义。
>
>然而，如果日志记录器 L 的一个祖先 P，设置了 additivity 标记为 false，那么 L 的输出L 的输出会被直接发送到 L 的所有 appender 和其祖先，包括 P。但是不会发送到 P 的appender。
>
>日志记录器默认设置了 additivity 属性为 true。


*下面的表格展示了一个例子：*

![8052a12116a6a90d7e85b11c810bcced.png](evernotecid://37F42873-4854-413F-861C-E51BA2619B6B/appyinxiangcom/22614073/ENResource/p5)

用户往往希望不仅可以定制输出目标，还可以定制输出格式。通过关联一个 `layout` 到 appender 可以实现格式化输出。Layout 负责根据用户配置对日志输出请求进行格式化，而 appender 负责发送格式化的日志到输出目标。PatternLayout 作为 logback 发布包的一部分，可以让用户用与 C 语言的 printf 函数类似的转换模式定制输出格式。

例如，指定转换模式为 "*%-4relative [%thread] %-5level %logger{32} - %msg%n*" 的 PatternLayout 会输出下面的的内容：

```text
176  [main] DEBUG manual.architecture.HelloWorld2 - Hello world.
```

第一个字段是从程序启动开始消耗的毫秒数。第二个字段是进行日志请求的线程名。第三个字段是日志请求的级别。第四个字段是与日志请求关联的日志记录器的名字。‘—’ 后的文本内容就是日志请求的消息。

#### 参数化日志输出

logback-classic 的日志记录器，实现了 SLF4J 的 Logger 接口，打印方法支持超过一个的参数。这些打印方法变体主要是为了最小化只读代码的影响来提升性能。

有一些日志记录器会这样打印，

```java
logger.debug("Entry number: " + i + " is " + String.valueOf(entry[i]));
```

将 integer i 和 entry[i] 转换为 String，并连接临时字符串，组装参数消息在是否进行实际日志输出都会产生无法忽略的开销。

一种可行的避免参数构造开销的方式是用一个判断来包裹日志声明。这是一个例子：

```java
if (logger.isDebugEnabled()) {
    logger.debug("Entry number: " + i + " is " + String.valueOf(entry[i]));
}
```

这种方式可以避免当 debugging 关闭时产生参数构造的开销。另一方面，如果日志记录器的 DEBUG 级别开启，日志记录器判断自身是否开启的开销会产生两次：一次在 debugEnabled 和一次在 debug。实际上，这些开销并不显著，因为判断一个日志记录器是否开启的开销少于实际输出日志请求消耗时间的 1%。


#### 更好的替代方式

存在一种更方便的基于消息格式化的替代方式。假设输入是一个对象，可以这样写：

```java
Object entry = new SomeObject();
logger.debug("The entry is {}.", entry);
```

只有在判断是否记录日志，并且当日志输出开启时，日志记录器实现才会格式化消息，以 entry 的字符串值替换 ‘{}’。换句话说，这种形式在日志声明关闭时不会产生参数构造开销。

下面两行代码会盛昌完全相同的输出。然而，在日志输出声明关闭时，第二种方式至少要比第一种方式提升 30% 的性能。

```java
logger.debug("The new entry is " + entry + ".");
logger.debug("The new entry is {}.", entry);
```

两个参数的变体也是可用。例如，可以编码：

```java
logger.debug("The new entry is {}. It replaces {}.", entry, oldEntry);
```

如果有三个或多余三个参数需要传递，一种 Object[] 的变体也是也已使用的。例如：

```java
Object[] paramArray = {newVal, below, above};
logger.debug("value {} was inserted between {} and {}.", paramArray);
```


#### 底层概览

再介绍了 logback 的核心组件之后，我们现在开始准备描述一下当用户调用了日志打印方法后， logback 框架执行的步骤。现在开始分析一下 logback 在用户调用了 “com.wombat” 日志记录器的 info() 方法后的步骤。

##### 1.获取过滤链判断

如果存在的话， TurboFilter 链会被调用。Turbo 过滤器可以设置上下文范围，或者根据如 Marker，Level，Logger 日志消息 或与每个日志请求关联的 Throwable 等的信息过滤确定的事件。如果过滤脸返回的是 FilterReply.DENY，日志请求会被丢弃，如果返回 FilterReply.NEUTRAL，会继续后续的步骤，如步骤 2。当返回是 FilterReply.ACCEPT，会跳过后续步骤直接执行步骤 3.

##### 2.应用基本命中规则

在该步骤中，logback 比较日志记录器的有效级别和日志请求的级别。如果日志请求在前面的判断中处于关闭，logback 不再进行其他处理，直接丢弃日志请求。否则，日志请求会被传递到下个步骤。

##### 3.创建日志输出事件对象

如果日志请求通过了之前的过滤，logback 会创建一个 "ch.qos.logback.classic.LoggingEvent" 对象，包含该请求所有相关参数，比如日志请求的日志记录器，请求级别，消息体，可能与请求一起传递的异常，当前时间，当前进程，一些关于日志请求和 MDC 的数据。注意这些信息之后在他们确实需要时才会进行初始化。MDC 是一些用来装饰日志请求的额外上下文信息。MDC 会在下一张进行讨论。

##### 4.调用 Appender

创建完 LoggingEvent 对象之后，logback 会调用从日志上下文中继承来的所有可用 appender 的  doAppend() 方法。

logback 发布包中的所有 appender 都扩展了 AppenderBase 抽象类，并在一个 synchronized 块中实现了 doAppend() 方法，保证线程安全。AppenderBase 的  doAppend() 方法也会调用与 appender 动态关联的自定义过滤器。自定义过滤器会在一个单独的章节进行说明。

##### 5.输出格式化

被调用的 appender 需要负责格式化日志事件。然而，一些（不是所有） appender 会将日志事件的格式化任务委托给一个 layout。一个 layout 格式化 LoggingEvent 示例后返回一个字符串作为结果。需要注意，一些 appender，像 SocketAppender 传递的是序列化的 LoggingEvent 实例，而不是字符串。因此，并不是所有的 appender 都需要一个 layout。

##### 6.发送日志输出事件

每个 appender 会把格式化后的日志事件发送到输出目标。

下面的 UML 图展示了所有阶段是如何工作的。你可能需要点击图片来显示一个更大的版本。

![](https://logback.qos.ch/manual/images/chapters/architecture/underTheHoodSequence2.gif)

### 性能

计算开销是经常引用的一个反对日志输出的论点。这是一个合理的考虑，即便是在大小适中的应用程序也会产生上千的日志请求。我们的很多开发工作会花费在 logbook 的性能测量和调优上。除了这些努力，用户仍应当关注下面这些性能问题。

##### 1.日志输出完全关闭时的性能

可以通过设置根日志记录器级别为最高级别 Level.OFF 来完全关闭日志输出。当日志输出完全关闭时，一个日志请求的开销有一个方法调用加上一个整型比较组成。在一台 3.2Ghz Pentium D 机器上这个开销大概是 20 纳秒。

然而，任何方法调用都可能包含“隐藏”的参数构造开销。例如：

```java
x.debug("Entry number: " + i + " is " + entry[i]);
```

会产生消息参数的构造的开销，如 integer i 和 entry[i] 的字符串转换和字符串的拼接，这些开销在日志开启或关闭时都会产生。

参数构造的开销根据参数大小会产生较高的开销。为了避免参数构造的开销，可以利用 SLF4J 参数化日志输出的优势：

```java
x.debug("Entry number: {} is {}", i, entry[i]);
```

这种变体不会产生参数构造的开销。与前一种 debug() 方法的调用相比，这种方式会大幅度加速。消息只在日志请求发送到 appender 后进行格式化。另外，消息格式化组件经过了高度的优化。

尽管将日志声明放置在 tight loops 中， 如频繁调用的代码，可能会导致性能降解。在 tight loops 中进行日志输出即便在日志输出关闭时，仍会产生大量（并且是无用的）输出。

##### 2.日志输出开启时判断日志是否输出的性能

在 logback 中，没有必要去遍历日志记录器的继承体系。一个日志记录器在创建的时候就已经确定了有效级别。一旦父日志记录器的级别发生改变，所有的子日志记录器都能够感知到变化。因此，在根据有效级别接受或拒绝前，日志记录器能够进行即时判断，不需要查询祖先。

##### 3.实际日志输出（格式化与写入输出设备）

这个开销是格式化日志输出与发送日志请求到目标输出。重新强调，大量的努力已经用来对格式化器性能进行了优化。同样也对 appender 进行了性能优化。实际输出日志到一个本地文件的大概开销是 9 ～ 12 毫秒。当输出日志到远程数据库时会增加几毫秒。

虽然拥有丰富的特性，除了可靠性，执行速度时 logback 设计的最重要目标。为了提升新能，logback 的一些组件已经经过了多次重写。

