---
layout: post
title: "利用 JUnit 的 Rule 对 Log4j 的输出进行测试"
date: 2014-08-14 22:07
comments: true
categories: [Programming]
---

最近在写框架的测试代码的时候，有需求要对 Log4j 的输出进行测试（**依赖 Log4j 的输出来进行测试，这一点本身可能得深思一下**），之前也有对 stdout 和 stderr 进行测试，用了一个叫做 `system-rule` 的包：

```xml
<dependency>
    <groupId>com.github.stefanbirkner</groupId>
    <artifactId>system-rules</artifactId>
    <version>1.5.0</version>
</dependency>
```

利用这个包中类，只需要在测试用例中加上一个 JUnit Rule，就可以获取到 stdout 和 stderr 中的内容，然后对其进行测试。现在我也想对 Log4j 的输出采用类似的方式进行测试，于是扩展了 JUnit 的 Rule，就有了以下这一段代码：

```java
/**
 * 此 Rule 用于对 Log4j 进行测试
 *
 * @author khotyn 8/14/14 9:18 PM
 */
public class Log4jRule extends ExternalResource {
    private String       logName;
    private List<String> loggerMessages = new ArrayList<String>();

    public Log4jRule(String loggerName) {
        this.logName = loggerName;
    }

    public Log4jRule(Class className) {
        this.logName = className.getName();
    }

    public List<String> getLoggerMessages() {
        return loggerMessages;
    }

    public String getLogMessageAsString() {
        String result = "";

        for (String loggerMessage : loggerMessages) {
            result += loggerMessage;
            result += "\n";
        }

        return result;
    }

    @Override
    protected void before() throws Throwable {
        Logger logger = LogManager.getLogger(logName);
        logger.addAppender(new AppenderSkeleton() {
            @Override
            protected void append(LoggingEvent event) {
                loggerMessages.add(event.getMessage().toString());
            }

            @Override
            public void close() {

            }

            @Override
            public boolean requiresLayout() {
                return false;
            }
        });
    }
}
```

整段代码非常简单，继承了 JUnit 的 `ExternalResource` 类，然后在 `before` 方法中，给对应的 Logger 加上了一个 Appender，在 Appender 中，将日志内容收集到一个 List 中，然后拿到这个 List 就可以拿到日志的输出了。

使用的时候非常简单：

```java
public class SampleTest {
    @Rule
    public Log4jRule log4jRule = new Log4jRule(SampleTest.class);

    @Test
    public void test() {
      	// .........
        Assert.assertTrue(log4jRule.getLogMessageAsString().contains(
            "Hello, world"));
    }
}
```

通过 

```java
log4jRule.getLogMessageAsString()
```

可以拿到一个 String 格式的日志输出，或者通过：


```java
log4jRule.getLoggerMessages()
```

来获得一个日志输出的内容的 List，List 中的没一行就是日志中的一行