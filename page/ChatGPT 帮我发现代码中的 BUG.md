今天一个很简单的功能触发了一个 BUG，处理后我想起了最近爆火的 ChatGPT，于是尝试测试 ChatGPT 能否发现这个 BUG。这篇文章会先介绍功能代码，然后手动分析 BUG 原因，接着测试 ChatGPT 能否发现这个 BUG。

---

## 前言

在测试一个准备上线的新功能时，发现了一段简单的函数出现了 BUG。以下是这个函数的功能描述：

**功能描述**  
- **输入**：一个文件夹路径。  
- **输出**：这个文件夹中的文件夹数量。

以下是初始实现的代码：

java
String pathString = "/Users/darcy/";
File fileDir = new File(pathString);
if (!fileDir.exists() || !fileDir.isDirectory()) {
    throw new RuntimeException("xxx....");
}
String[] fileList = fileDir.list();
int count = 0;
for (String filePath : fileList) {
    if (new File(pathString + filePath).isDirectory()) {
        count++;
    }
}
System.out.println(count);


功能实现了，但代码较为繁琐，于是优化为以下版本：

java
Path path = Paths.get("/Users/darcy/");
if (!Files.exists(path) || !Files.isDirectory(path)) {
    throw new RuntimeException("xxx....");
}
long dirCount = Files.list(path).filter(Files::isDirectory).count();
System.out.println(dirCount);


优化后的代码更简洁，但正是这段代码触发了 BUG。你能看出问题在哪里吗？

---

## 问题分析

优化后的代码在少量运行时表现正常，但存在一个问题：**`Files.list` 返回的 Stream 流没有关闭**。如果这段代码被频繁调用，长时间运行后会因为打开的文件过多而报错。以下是问题的复现代码：

java
while (true) {
    Path path = Paths.get("/Users/darcy/");
    if (!Files.exists(path) || !Files.isDirectory(path)) {
        throw new RuntimeException("xxx....");
    }
    long dirCount = Files.list(path).filter(Files::isDirectory).count();
    System.out.println(dirCount);
}


运行一段时间后，控制台会报错：**Too many open files**。

### 问题原因

每个进程可以打开的文件数量是有限的。如果一直打开文件而不关闭，达到限制时会报错。可以通过以下代码监控进程持有的文件数：

bash
# 查找进程 ID
➜  ~ jps -mlVv | grep testFileList
# 查看指定进程引用的文件数量
➜  ~ lsof -p 62248 | wc -l


测试结果显示，随着程序运行，进程持有的文件数量不断上升。

---

## 问题解决

既然问题是 `Files.list` 返回的 Stream 流没有关闭，那么只需在代码中显式关闭流即可。以下是两种解决方案：

### 方式 1：手动关闭流

java
while (true) {
    Path path = Paths.get("/Users/darcy/");
    if (!Files.exists(path) || !Files.isDirectory(path)) {
        throw new RuntimeException("xxx....");
    }
    Stream<Path> pathStream = Files.list(path);
    long dirCount = pathStream.filter(Files::isDirectory).count();
    pathStream.close();
    System.out.println(dirCount);
    Thread.sleep(1000);
}


### 方式 2：使用 `try-with-resources` 自动关闭流

java
while (true) {
    Path path = Paths.get("/Users/darcy/");
    if (!Files.exists(path) || !Files.isDirectory(path)) {
        throw new RuntimeException("xxx....");
    }
    try (Stream<Path> pathStream = Files.list(path)) {
        long dirCount = pathStream.filter(Files::isDirectory).count();
        System.out.println(dirCount);
    }
    Thread.sleep(1000);
}


再次运行后，引用的文件数量不再上涨，BUG 解决。

---

## ChatGPT 分析

如果将这段代码交给 ChatGPT，它能发现问题吗？答案是肯定的。以下是 ChatGPT 的分析结果：

### ChatGPT 分析代码

**提问**：下面这段 Java 代码有什么问题吗？

java
Path path = Paths.get("/Users/darcy/");
if (!Files.exists(path) || !Files.isDirectory(path)) {
    throw new RuntimeException("fileDir.isDirectory() error");
}
long dirCount = Files.list(path).filter(Files::isDirectory).count();
System.out.println(dirCount);


**ChatGPT 回答**：  
这段代码看起来没有语法错误，但存在以下问题：  
1. **资源泄漏**：`Files.list` 返回的 Stream 流需要关闭，否则可能导致资源泄漏。建议使用 `try-with-resources` 自动关闭流。  
2. **异常处理**：对文件系统操作时可能抛出 `IOException`，建议捕获异常或在方法签名中声明抛出异常。

修复代码示例：

java
try (Stream<Path> stream = Files.list(path)) {
    long dirCount = stream.filter(Files::isDirectory).count();
    System.out.println(dirCount);
} catch (IOException e) {
    // 处理异常
}


---

## 总结

本文介绍了一个简单功能的 Java 代码 BUG，即未关闭 `Files.list()` 返回的流对象，导致程序运行时可能会打开过多的文件而出现异常。通过手动分析代码和测试，找出了问题所在，并提供了两种解决方案：手动关闭流和使用 `try-with-resources` 自动关闭流。

此外，我们还测试了 ChatGPT 是否可以发现代码中的问题，结果表明 ChatGPT 能够准确识别问题并提供修复建议。这表明，ChatGPT 可以成为编程工作中的得力助手。

---

👉 [WildCard | 一分钟注册，轻松订阅海外线上服务](https://bit.ly/bewildcard)
