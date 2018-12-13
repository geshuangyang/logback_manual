# 第二单元：架构

## Logback's 架构

Logback 的基础架构非常通用，因此适用于很多不同的场景。下面将分为三个模块来说明，`logback-core`，`logback-classic`,`logback-access`。

`core` 模块是另外两个模块的底层依赖。`classic` 模块扩展了 `core` 模块。`classic` 模块与 `log4j` 相比较有非常显著的改进。`logback-classic` 原生实现了 SLF4J API，因此可以方便的在 logback 和其他日志系统间进行切换，比如 log4j 或 在 JDK1.4 开始引入的 `java.util.logging`(JUL)。第三个模块 access 集成了 Servlet 容器，以此来提供 HTTP 访问日志的功能。access 模块会通过一个独立的文档进行说明。

在本章节的剩下部分，我们会用 logback 来指代 logback-classic 模块。


## Logger，Appenders 与 Layouts

Logback 的实现基于三个核心的类：Logger，Appender 和 Layout。这三个组件共同工作是开发者能够根据消息类型和级别进行日志输出，并在运行时控制消息的格式化与报告。

`Logger` 类是 logback-classic 模块的一部分。另一方面，Appender 和 Layout 接口是 logback-core 的一部分。作为一个通用的模块，logback-core 没有日志记录器（logger）的概念。


## Logger 上下文

基于 System.out.print 的 日志输出 API 最重要的优势就是允许用户自由的打印或是关闭日志声明的能力。这个功能的前提是假设日志空间（所有可能的日志输出声明的空间）以开发者选择的标准进行了分类。在 logback-classic 中，这种分类方式是日志记录器的固有组成部分。每个单独的日志记录器都会与一个 LoggerContext 进行关联，LoggerContext 负责生成大量的日志记录器，并将这些日志记录器以树形结构进行组织。

日志记录器都是命名实体。他们的名字大小写敏感，同时遵循集成命名规则：


## 命名体系

一个日志记录器的名字附加上 `.` 组成的字符串如果是其他日志记录器名字的前缀，就认为这个日志记录器是这些日志记录器的祖先。如果一个日志记录器与后代日志记录器之间没有其他日志记录器，就成这个日志记录器是后代日志记录器的父日志记录器。

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


## 有效级别又称级别继承

日志记录器可以分配级别。级别集合（TRACE，DEBUG，INFO，WARN，ERROR）在 `ch.qos.logback.classic.Level` 类中定义。在 logback 中，`Level` 是 final 类，无法被扩展子类，相比之前的 Marker 对象更加灵活。

如果一个日志记录器没有分配级别，会继承最近的已分配级别的祖先的级别。更正式的表述：

```text
一个给定的日志记录器 L 的有效级别，是在以 L 自身并向上直至根日志记录器的继承体系中第一个非 null 的级别。
```

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


## 打印方法和基本的选择规则


日志打印方法在定义中确定了一个日志输出请求的日志级别。例如，如果 L 是一个日志记录器实例，则 L.info("..“) 就是一个日志级别为 INFO 的日志输出声明。

如果一个日志输出请求的级别等于或高于日志记录器的有效级别，就说该日志输出请求是激活的。否则，该日志输出请求是无效的。如之前的描述，一个没有分配级别的日志记录器会继承最近祖先的日志级别。这条规则在下面进行了总结。


## 基本命中规则

```text
如果一个级别为 *p* 日志请求非分发给了一个有效级别为 *q* 的日志记录器，则当 *p >= q* 时，日志请求生效。
```

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
           
           
## 调用日志记录器


## Appenders 与 Layouts

#### Appender Additivity


## 参数化日志输出

#### 更好的替代方式


## 底层概览

#### 1.获取过滤链判断
#### 2.应用基本命中规则
#### 3.创建日志输出事件对象
#### 4.调用 Appender
#### 5.输出格式化
#### 6.发送日志输出事件


## 性能

#### 1.日志输出完全关闭时的性能
#### 2.日志输出开启时判断日志是否输出的性能
#### 3.实际日志输出（格式化与写入输出设备）

