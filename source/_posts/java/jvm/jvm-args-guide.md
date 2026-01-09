---
title: JVM参数
date: 2025-12-01
desc: JVM参数
keywords: jvm
categories: [ JVM ]
---

# 📘 JVM 启动参数前缀详解（-D / -X / -XX）

> **适用版本**：OpenJDK / Oracle JDK（HotSpot JVM）  
> **最后更新时间**：2026年1月  
> **目标读者**：Java 开发者、SRE、性能工程师

---

## 一、`-D`：设置系统属性（System Properties）

### ✅ 含义
用于在 JVM 启动时定义 **Java 系统属性（System Properties）**。这些属性属于应用程序级别的配置，可通过 `System.getProperty()` 在代码中读取。

### 🔧 语法
```bash
-Dproperty_name=value
```
> ⚠️ 注意：`-D` 与属性名之间 **不能有空格**，否则会被视为无效参数。

### 📌 常见用途
- 配置文件路径
- 指定运行环境（如 Spring profile）
- 覆盖默认行为（如字符编码、时区）

### 🛠 常用参数示例

| 参数 | 说明 |
|------|------|
| `-Dfile.encoding=UTF-8` | 设置默认字符编码为 UTF-8 |
| `-Duser.timezone=Asia/Shanghai` | 设置默认时区 |
| `-Dspring.profiles.active=prod` | Spring Boot 激活生产环境配置 |
| `-Dlogback.configurationFile=/conf/logback.xml` | 指定 Logback 配置文件 |
| `-Djava.security.egd=file:/dev/./urandom` | 加快 SecureRandom 初始化（常用于容器环境） |

### 💡 使用示例
```bash
java -Dfile.encoding=UTF-8 -Denv=production -jar app.jar
```

在 Java 代码中读取：
```java
String env = System.getProperty("env"); // 返回 "production"
```

---

## 二、`-X`：JVM 非标准扩展选项（Non-Standard Options）

### ✅ 含义
由 JVM 实现（如 HotSpot）提供的 **非标准但广泛支持的扩展参数**，主要用于控制内存、线程栈、GC 日志等运行时行为。

> 虽然“非标准”，但在 OpenJDK/Oracle JDK 中几乎成为事实标准。

### 🔧 查看所有 `-X` 选项
```bash
java -X
```

### 🛠 常用参数示例

| 参数 | 说明 |
|------|------|
| `-Xmx<size>` | 设置 JVM **最大堆内存**（如 `-Xmx4g`） |
| `-Xms<size>` | 设置 JVM **初始堆内存**（如 `-Xms2g`） |
| `-Xss<size>` | 设置 **每个线程的栈大小**（如 `-Xss1m`，默认通常 1M） |
| `-Xmn<size>` | 设置 **年轻代（Young Generation）大小**（较少用，推荐用 `-XX:NewSize`） |
| `-Xloggc:<file>` | （JDK 8 及更早）将 GC 日志输出到文件（JDK 9+ 推荐用 `-Xlog`） |
| `-Xdebug` | 启用调试模式（旧版，现代用 `-agentlib:jdwp`） |

### 💡 使用示例
```bash
java -Xms2g -Xmx4g -Xss512k -jar myapp.jar
```

> 📌 建议：`-Xms` 和 `-Xmx` 设为相同值可避免堆动态扩容带来的性能抖动。

---

## 三、`-XX`：高级/实验性 JVM 选项（Advanced / Experimental）

### ✅ 含义
用于精细控制 **JVM 内部行为**，包括垃圾回收器选择、JIT 编译策略、内存管理、诊断工具等。这些选项 **高度依赖 HotSpot JVM**，不同 JDK 版本差异较大。

### 🔧 两类格式
1. **布尔型**：  
   - `-XX:+OptionName` → 启用  
   - `-XX:-OptionName` → 禁用  
2. **键值型**：  
   - `-XX:OptionName=value`

### 🛠 常用参数分类汇总

#### 🗑 1. 垃圾回收（GC）调优
| 参数 | 说明 |
|------|------|
| `-XX:+UseG1GC` | 使用 G1 垃圾回收器（JDK 9+ 默认） |
| `-XX:+UseParallelGC` | 使用并行 GC（吞吐量优先，JDK 8 默认） |
| `-XX:+UseZGC` | 启用 ZGC（低延迟，JDK 15+ 生产就绪） |
| `-XX:MaxGCPauseMillis=200` | 设置 GC 最大暂停时间目标（G1/ZGC 有效） |
| `-XX:G1HeapRegionSize=16m` | 设置 G1 Region 大小（1~32MB，需 2 的幂） |

#### 🧠 2. 内存管理
| 参数 | 说明 |
|------|------|
| `-XX:MetaspaceSize=256m` | 元空间触发 GC 的初始阈值 |
| `-XX:MaxMetaspaceSize=512m` | 限制元空间最大内存（防 OOM） |
| `-XX:+UseCompressedOops` | 启用压缩普通对象指针（节省内存，默认开启） |
| `-XX:+AlwaysPreTouch` | 启动时预分配并清零堆内存（减少运行时缺页中断） |

#### 📊 3. 诊断与监控
| 参数 | 说明 |
|------|------|
| `-XX:+HeapDumpOnOutOfMemoryError` | OOM 时自动生成堆转储 |
| `-XX:HeapDumpPath=/data/dumps/` | 指定堆转储文件路径 |
| `-XX:+PrintGCDetails` | （JDK 8）打印详细 GC 日志 |
| `-XX:+PrintGCDateStamps` | 在 GC 日志中添加时间戳 |
| `-XX:OnError="sh /scripts/alert.sh %p"` | JVM 崩溃时执行脚本 |

> 💡 **JDK 9+ 新日志系统替代旧 GC 日志**：
> ```bash
> -Xlog:gc*:file=gc.log:time,tags,level
> ```

#### ⚙️ 4. JIT 与性能
| 参数 | 说明 |
|------|------|
| `-XX:+TieredCompilation` | 启用分层编译（C1 快速编译 + C2 优化编译） |
| `-XX:TieredStopAtLevel=1` | 仅使用 C1 编译（降低 CPU，用于调试或低负载） |
| `-XX:ReservedCodeCacheSize=240m` | 设置 JIT 编译代码缓存大小 |

---

## 四、对比总结表

| 前缀 | 类型 | 是否标准 | 主要用途 | 是否推荐生产使用 |
|------|------|--------|--------|----------------|
| `-D` | 系统属性 | ✅ 是（Java 标准） | 应用配置、环境变量 | ✅ 强烈推荐 |
| `-X` | 扩展选项 | ❌ 否（JVM 实现相关） | 堆内存、线程栈、GC 日志 | ✅ 广泛使用 |
| `-XX` | 高级选项 | ❌ 否（HotSpot 特有） | GC 调优、JIT、诊断 | ⚠️ 谨慎使用，需充分测试 |

---

## 五、最佳实践建议

1. **优先使用 `-D` 传递应用配置**，而非硬编码。
2. **固定堆大小**：生产环境建议 `-Xms = -Xmx`，避免动态扩容开销。
3. **GC 选择**：
   - 低延迟 → G1（JDK 8u40+）或 ZGC/Shenandoah（JDK 11+）
   - 高吞吐 → Parallel GC
4. **务必开启 OOM 堆转储**：
   ```bash
   -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/logs/
   ```
5. **避免盲目复制网上的 `-XX` 参数**：不同应用、JDK 版本效果可能截然相反。
6. **使用 JDK 自带工具验证**：
   - `jstat -gc <pid>`：实时监控 GC
   - `jinfo -flags <pid>`：查看当前生效的 JVM 参数

---

## 六、附录：快速查看 JVM 参数

```bash
# 查看所有可管理的 JVM 参数（含默认值）
java -XX:+PrintFlagsFinal -version | grep manageable

# 查看当前进程的 JVM 启动参数
jcmd <pid> VM.flags
```