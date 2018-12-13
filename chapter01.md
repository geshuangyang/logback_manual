# Logback 简介

## 什么是 logback ？

Logback 可以认为是 log4j 项目的继任者，由 log4j 的创建者 Ceki Gülcü 设计，基于大量的工业级强度的日志系统的设计经验建设。Logback 是目前现存的日志系统中最快的，并且占用更小的计算资源，偶尔会有大幅提高。另外同样重要的是 logback 提供其他日志系统不具有的有用特性。

## 第一步
Logback-classic 模块需要在 classpath 同时引入 slf4j-api.jar、logbook-core.jar 和 logback-classic.jar。
Logback-*.jar 文件是 logback 的一部分，而slf4j-api.jar 是 SLF4J 的一部分，两者是独立的项目。

现在让我们开始体验一下 logback 的使用。

示例 1.1 基本日志输出模版

```java
    package chapters.introduction;

    import org.slf4j.Logger;
    import org.slf4j.LoggerFactory;

    public class HelloWorld1 {
        public static void main(String[] args) {

            Logger logger = LoggerFactory.getLogger(“chapter.introduction.HelloWorld1”);
            logger.debug(“Hello World.”);

        }
    }
```

HelloWorld1 类由包 chapters.introduction 定义，在开头引入了包 org.slf4j 的 SLF4J API 定义的 Logger 和 LoggerFactory 类。

在 main() 方法的第一行，通过调用类 LoggerFactory 的 静态方法 getLogger() 获取了一个 Logger 示例，并将其赋给变量 logger。这个 logger 被命名为“chapter.introduction.HelloWorld1”。main 方法接下来以 “Hello World ” 作为方法参数调用了 logger 的 debug 方法。因此 main 方法包含了一个级别为 DEBUG，消息为 “Hello World” 的日志输出声明。

在上面的示例中并没有引用任何 logback 的类。大多数情况就日志输出而言，你只需要引入 SLF4J 的类。因此，如果程序中绝大多数的类使用 SLF4J API 可以无视 logback 的存在。

可以通过下面的命令运行第一个简单程序，chapters.introduction.HelloWorld1:

```shell
    Java chapters.introduction.HelloWorld1
```

运行 HelloWorld1 会在终端输出一行日志信息。当未获取到默认配置文件，logback的默认配置策略会添加一个 ConsoleAppender 到 root logger。

```text
    20:49:07.962 [main] DEBUG chapters.introduction.HelloWorld1 - Hello world.
```

使用 Logback 的内置状态系统可以获取到其内部的状态信息。通过组件 StatusManager 可以获取 logback 声明周期中发生的重要事件。首先，我们先调用 StatusPrinter 类的 print() 方法来打印 logback 的内部状态。
 

示例 1.2：打印 Logger 状态(logback-examples/src/main/java/chapters/introduction/HelloWorld2.java)

```java
    package chapters.introduction;

    import org.slf4j.Logger;
    import org.slf4j.LoggerFactory;
    import ch.qos.logback.classic.LoggerContext;
    import ch.qos.logback.core.util.StatusPrinter;

    public class HelloWorld2 {
        public static void main(String[] args) {
            Logger logger = LoggerFactory.getLogger(“chapters.introduction.HelloWorld2”);
            logger.debug(“Hello World.”);

            // 打印内部状态
            LoggerContext lc = (LoggerContext) LoggerFactory.getILoggerFactory();
            StatusPrinter.print(lc);
        }
    }
```



运行 HelloWorld2 程序将产生下列输出：

```text
    12:49:22.203 [main] DEBUG chapters.introduction.HelloWorld2 - Hello world.
    12:49:22,076 |-INFO in ch.qos.logback.classic.LoggerContext[default] - Could NOT find resource [logback.groovy]
    12:49:22,078 |-INFO in ch.qos.logback.classic.LoggerContext[default] - Could NOT find resource [logback-test.xml]
    12:49:22,093 |-INFO in ch.qos.logback.classic.LoggerContext[default] - Could NOT find resource [logback.xml]
    12:49:22,093 |-INFO in ch.qos.logback.classic.LoggerContext[default] - Setting up default configuration.
```

Logback 说明了获取 logback-test.xml 和 logback.xml 配置文件失败（稍后讨论），然后通过默认策略进行配置，也就是一个基本的 ConsoleAppender。Appender 可以看作是一个目标输出的类。Appenders 存在很多不同的目标输出，包括终端，文件，Syslog， TCP Sockets， JMS 和其他。用户也可以很容易的创建自定义的Appenders。

当发现错误的情况时，logback 会自动在终端打印内部状态。

上面的示例相对很简单。实际上在一个大型应用程序中日志输出会很不同。日志输出声明的大致模式并不会改拜年。只有配置处理会有差异。然而，对于不同的需求，你可能会需要定制和配置 logback。Logback 配置会在下个章节进行说明。

在上面的示例中，我们已经通过调用 StatusPrinter.print() 方法委托 logback 打印了内部状态。Logback 的内部状态信息对于诊断 logback 相关的问题时会非常有用。

下面列表展示了在应用程序中开启日志输出的三个必须步骤。
  1. 配置 logback 环境。可以通过很多方式完成，稍后会详细说明。
  2. 在所有需要输出日志的类中，通过调用 org.slf4j.LoggerFactory 类来获取一个 Logger 示例，传递当前类名或类本身作为一个参数。
  3. 使用第二步获取的 Logger 实例的打印方法，debug()，info()，warn()，error()。这些日志会通过配置的 Appenders 进行输出。

  
## 构建 Logback

Logback 依赖广泛使用的开源构建工具 Maven。

一旦你安装了 Maven ，构建 logback 工程与相关模块只需要在解压后的 logback 发布包目录下运行 mvn install 即可。Maven 会自动下载需要的外部依赖库。

Logback 发布包包含完整的源码，所有可以修改logback的库构建用户自己的版本。你也可以重新发布修改后的版本，只要同样遵守 LGPL 整数或者 EPL 证书的条件。

为了在 IDE 下构建 logback，请参考类路径下的相关文件说明。





