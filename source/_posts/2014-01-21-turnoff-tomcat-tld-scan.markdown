---
layout: post
title: "关闭 Tomcat 的 TLD 扫描的功能"
date: 2014-01-21 19:12
comments: true
categories: [Programming, Java, Tomcat]
---

### 背景

Tomcat 作为 Servlet 规范的实现者，它在应用启动的时候会扫描 Jar 包里面的 .tld 文件，加载里面定义的标签库，但是，我们在开发的时候很多都不是采用 JSP 作为 Web 页面的模板的，很多都是使用  Velocity 之类的模板引擎，自然而然，为了加快应用的启动速度，我们可以把 Tomcat 里面的这个功能给关掉。

### 方法

看 Tomcat 的配置文档，关于 Context 的设置这一块，看到了 `processTlds` 这个属性可以设置，看下这个属性的说明：

> Whether the context should process TLDs on startup. The default is true. The false setting is intended for special cases that know in advance TLDs are not part of the webapp.

只要在 Context 中把这个属性设置成 false，那么我们就可以关闭 Tomcat 的 TLD 扫描功能了，为了让所有的应用都可以关闭这个功能，我们可以将 Tomcat 目录下的 conf/context.xml 修改成如下这样：


```
<?xml version='1.0' encoding='utf-8'?>
<Context processTlds="false">
    <WatchedResource>WEB-INF/web.xml</WatchedResource>
</Context>

```

#### 坑

但是，在 Tomcat 6 中测试的时候，发现这个功能没有生效，无奈只能 Debug Tomcat 的源码，发现 StandardContext 的 init 方法下有如下代码：

```
if (processTlds) {
    this.addLifecycleListener(new TldConfig());
}

super.init();

// Notify our interested LifecycleListeners
lifecycle.fireLifecycleEvent(INIT_EVENT, null);
```

这里需要说明的一点是，我们的默认的 context 配置是在 `lifecycle.fireLifecycleEvent(INIT_EVENT, null);` 这行代码中被处理的，而在这行代码之前，Tomcat 就已经使用了 `processTlds`，我们的配置完全没有生效。

#### Workaround

那么，这么解决呢？在 context 中，我们还可以配置一个 JarScanner，这个 JarScanner 会被用来扫描 Jar 包中的 tld 文件，我们可以在默认的 context.xml 中配置一个空的 JarScanner，像下面这样：

```
<?xml version='1.0' encoding='utf-8'?>
<Context processTlds="false">
    <JarScanner className="com.alipay.sofa.runtime.test.patch.tomcat.NullJarScanner"/>
</Context>
```

NullJarScanner 的代码如下：

```
package com.alipay.sofa.runtime.test.patch.tomcat;

import org.apache.tomcat.JarScanner;
import org.apache.tomcat.JarScannerCallback;

import javax.servlet.ServletContext;
import java.util.Set;

/**
 * @author khotyn 14-1-21 下午4:37
 */
public class NullJarScanner implements JarScanner {
    @Override
    public void scan(ServletContext context, ClassLoader classloader, JarScannerCallback callback, Set<String> jarsToSkip) {
        // Do nothing at all.
    }
}

```

**需要注意的是，Tomcat 7 不会出现上述的问题，你只要在配置中把 processTlds 设置成 false 即可。**