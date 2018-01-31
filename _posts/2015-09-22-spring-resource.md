---
layout: post
title: "Spring Resource"
categories: 分布式系统
tags: [Spring, Resource]
---

JDK所提供的访问资源的类（如java.net.URL、File等）并不能很好地满足各种底层资源的访问需求，不如缺少从类路径或者Web容器上下文中获取资源的操作类。因此，Spring设计了Resource接口抽象了底层资源的访问。

## Resource 接口

```java
public interface Resource extends InputStreamSource {

  boolean exists();

  boolean isOpen();

  URL getURL() throws IOException;

  File getFile() throws IOException;

  Resource createRelative(String relativePath) throws IOException;

  String getFilename();

  String getDescription();
}
```

```java
public interface InputStreamSource {
  InputStream getInputStream() throws IOException;
}
```

主要方法有：

- `getInputStream()`：返回资源对应的输入流。每次调用都会返回一个全新的`InputStream`，并依赖调用者手动关闭。
- `exists()`：返回资源是否存在。
- `isOpen()`：返回资源是否打开。
- `getDescription()`：返回资源描述标识。通常是URL的全限定名，或者是文件的全路径。

## 内置Resource实现

![Spring Resource UML]({{ site.baseurl | append: "/images/spring-resource/spring-resource-uml.png" }})

常用实现类如下：

- `WritableResource`：可写资源接口，有两个实现类，FileSystemResource和PathResource。
- `ByteArrayResource`：二进制数组标识的资源。
- `ClassPathResource`：类路径下的资源。
- `FileSystemResource`：文件系统标识的资源。
- `InputStreamResource`：以输入流标识的资源。
- `URLResource`：封装了`java.net.URL`，可以使用户访问
- `ServletContextResource`：为访问Web容器上下文而设计的类，负责以相对于Web应用根目录的路径加载资源。它支持以流和URL的方式，在WAR包解压的情况下，还支持以File形式访问。
- `PathResource`：封装了`java.nio.file.Path`、`java.net.URL`、系统文件资源，它使用户可以访问任何通过URL、Path、系统文件路径表示的资源。

## 资源加载

访问不同的资源需要不同的`Resource`实现类，这是比较麻烦的。Spring提供了一个比较强大的资源家在机制，可以使用`classpath:`、`file:`等资源前缀通配符访问不同类型的方式。

### 资源地址表达式

| 地址前缀       | 示例                             | 对应的资源类型                                  |
| ---------- | ------------------------------ | ---------------------------------------- |
| classpath: | classpath:com/myapp/config.xml | 使用ClassPathResource从classpath加载。         |
| file:      | file:///data/config.xml        | 使用UrlResource从文件系统加载。                    |
| http://    | http://myserver/logo.png       | 使用UrlResource加载。                         |
| ftp://     | ftp://myserver/file            | 使用UrlResource加载。                         |
| 没有前缀       | /data/config.xml               | 根据使用的ApplicationContext决定，如使用 ClassPathXmlApplicationContext则返回 ClassPathResource实例；如使用 FileSystemXmlApplicationContext则是 FileSystemResource；如 WebApplicationContext则是 ServletContextResource。 |

和`classpath:`的类似的有一个`classpath*:`变体，两者的区别是前者只会在第一个加载的`com.myapp`包下搜索，而后者会扫描所有类路径下出现的`com.myqpp`包。

Ant风格的资源地址支持三种通配符：

- `?`：匹配一个字符
- `*`：匹配文件名的任意长度字符
- `**`：匹配多层路径

如下示例：

```
/WEB-INF/*-context.xml
com/mycompany/**/applicationContext.xml
file:C:/some/path/*-context.xml
classpath:com/mycompany/**/applicationContext.xml
```

### 资源加载器

Spring定义了一套资源加载的接口和实现：

![Spring ResourceLoader UML]({{ site.baseurl | append: "/images/spring-resource/spring-resourceloader-uml.png" }})

- `ResourceLoader`：仅有一个`getResource`方法，**不支持Ant风格通配符**。
- `ResourcePatternResolver`：定义了`getResources`方法，支持Ant通配符。
- `ResourceLoaderAware`：类似`ApplicationContextAware`，在bean实例化时提供一个`ResourceLoader`实例。Spring所有的`ApplicationContext`都实现了`ResourcePatternResolver`接口，实际上`ResourceLoaderAware`提供的就是`ApplicationContext`实例。因此可以使用`ApplicationContextAware`替代`ResourceLoaderAware`接口。
- `PathMatchingResourcePatternResolver`：Spring提供的标准实现类。

```java
ResourcePatternResolver resolver = new PathMatchingResourcePatternResolver();
Resource[] resources = resolver.getResources("classpath*:com/myapp/*.xml");
```

## 参考资料

- [Spring Core Technologies](https://docs.spring.io/spring/docs/current/spring-framework-reference/core.html#resources-introduction)