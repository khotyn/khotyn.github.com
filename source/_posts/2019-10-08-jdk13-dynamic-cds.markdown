---
layout: post
title: "JDK13 新特性之 Dynamic CDS"
date: 2019-10-08 20:51
comments: true
categories: 
---

一年多以前，我写过一篇文章[Java 10 新特性之 AppCDS](/blog/2018/03/21/app-cds/)，文章的最后有一个结论：

> 虽然 AppCDS 号称可以支持自定义的 Classloader，但是我试了一个 SpringBoot 的应用，发现对于没有在 -classpath 中指定的 JAR 包中的类，并不会有效果。

在 Java 的世界中，自定义的 Classloader 情况太多了，这个大大限制了 AppCDS 的应用，不过，这次看了 JDK13 的 Release Note，很开心看到 JDK13 对 CDS 的功能进行了增强，本次对 CDS 的增强主要是两个方面：

* 一个是简化了 CDS 的使用，在原来的步骤中，需要生成能够 dump 的 Class 文件的列表，然后再根据这份文件生成 dump 的内容，然后再使用 dump 的内容进行启动的加速。现在一步就可以直接生成 dump 文件了拿过来做启动的加速了，比原来少了一步。
* 另一个增强是现在 CDS 不仅仅会 dump `-cp` 指定的类路径下的 Class，并且会 dump 在应用退出之前所有已经加载的类，有个这个特性，CDS 就能够用在各种场景下了。

接下来，我们就尝试一下在 SOFABoot 中使用 Dynamic CDS，首先新建一个 Spring Boot 的应用，并且把 parent 替换为：

```xml
<parent>
    <groupId>com.alipay.sofa</groupId>
    <artifactId>sofaboot-dependencies</artifactId>
    <version>3.1.5</version>
</parent>
```

然后我们可以编译并且启动一下，看下耗时情况：

```
➜  time java -jar target/dynamiccds-0.0.1-SNAPSHOT.jar 

  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::        (v2.1.0.RELEASE)

2019-10-08 23:16:55.768  INFO 69815 --- [           main] c.e.dynamiccds.DynamiccdsApplication     : Starting DynamiccdsApplication v0.0.1-SNAPSHOT on lotus.local with PID 69815 (/Users/khotyn/Downloads/dynamiccds/target/dynamiccds-0.0.1-SNAPSHOT.jar started by khotyn in /Users/khotyn/Downloads/dynamiccds)
2019-10-08 23:16:55.772  INFO 69815 --- [           main] c.e.dynamiccds.DynamiccdsApplication     : No active profile set, falling back to default profiles: default
2019-10-08 23:16:56.377  INFO 69815 --- [           main] c.e.dynamiccds.DynamiccdsApplication     : Started DynamiccdsApplication in 0.981 seconds (JVM running for 1.411)
java -jar target/dynamiccds-0.0.1-SNAPSHOT.jar  4.14s user 0.29s system 301% cpu 1.473 total
```

耗时时间是 4.14s。

然后我们用如下的命令生成一下 Class 的 Dump：

```
java -XX:ArchiveClassesAtExit=dcds.jsa -jar target/dynamiccds-0.0.1-SNAPSHOT.jar
```

然后我们再执行以下命令就可以使用刚才 Dump 出来的文件了：

```
time java -XX:SharedArchiveFile=dcds.jsa -jar target/dynamiccds-0.0.1-SNAPSHOT.jar
```

可以看下输出：

```
➜  time java -XX:SharedArchiveFile=dcds.jsa -jar target/dynamiccds-0.0.1-SNAPSHOT.jar

  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::        (v2.1.0.RELEASE)

2019-10-08 23:20:44.078  INFO 70738 --- [           main] c.e.dynamiccds.DynamiccdsApplication     : Starting DynamiccdsApplication v0.0.1-SNAPSHOT on lotus.local with PID 70738 (/Users/khotyn/Downloads/dynamiccds/target/dynamiccds-0.0.1-SNAPSHOT.jar started by khotyn in /Users/khotyn/Downloads/dynamiccds)
2019-10-08 23:20:44.083  INFO 70738 --- [           main] c.e.dynamiccds.DynamiccdsApplication     : No active profile set, falling back to default profiles: default
2019-10-08 23:20:44.636  INFO 70738 --- [           main] c.e.dynamiccds.DynamiccdsApplication     : Started DynamiccdsApplication in 0.91 seconds (JVM running for 1.462)
java -XX:SharedArchiveFile=dcds.jsa -jar target/dynamiccds-0.0.1-SNAPSHOT.jar  2.94s user 0.30s system 210% cpu 1.540 total
```

耗时是 2.94s，时间快了将近一秒多，这个时间可能相比于有大量业务逻辑的应用来说意义不大，但是也算是非常可观了。

