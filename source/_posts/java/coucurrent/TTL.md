这是一个关于如何配置和使用 Alibaba `transmittable-thread-local` (TTL) Java Agent 的规整 Markdown 文档。

---

# Transmittable Thread Local (TTL) Java Agent 配置指南

在多线程编程中，为了在线程池中实现 `ThreadLocal` 变量的正确传递，推荐使用 TTL 的 Java Agent 模式。这种方式可以在不修改业务代码（无侵入）的情况下，自动完成线程间的上下文传递。

## 1. Maven 依赖配置

首先，在 `pom.xml` 中引入 TTL 依赖，并定义版本号。

```xml
<properties>
    <transmittable.version>2.14.5</transmittable.version>
</properties>

<dependencies>
    <dependency>
        <groupId>com.alibaba</groupId>
        <artifactId>transmittable-thread-local</artifactId>
        <version>${transmittable.version}</version>
    </dependency>
</dependencies>
```

## 2. 构建插件配置 (自动拷贝 Agent)

为了方便打包和部署，使用 `maven-dependency-plugin` 在 `package` 阶段将 TTL 的 JAR 包拷贝到指定目录（如 `target/agent`），并重命名为 `ttl-agent.jar`。

请将以下配置添加到 `build.plugins` 节点下：

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-dependency-plugin</artifactId>
    <executions>
        <execution>
            <id>copy-ttl-agent-jar</id>
            <phase>package</phase>
            <goals>
                <goal>copy</goal>
            </goals>
            <configuration>
                <artifactItems>
                    <artifactItem>
                        <groupId>com.alibaba</groupId>
                        <artifactId>transmittable-thread-local</artifactId>
                        <version>${transmittable.version}</version>
                        <type>jar</type>
                        <destFileName>ttl-agent.jar</destFileName>
                    </artifactItem>
                </artifactItems>
                <outputDirectory>${project.build.directory}/agent</outputDirectory>
            </configuration>
        </execution>
    </executions>
</plugin>
```

## 3. 本地启动方式

在本地运行项目时，需要在 JVM 启动参数中加入 `-javaagent` 指令。

> **注意：** 如果不加 Agent 启动，在线程池场景下可能会丢失线程上下文。

```bash
java \
  -javaagent:target/agent/ttl-agent.jar \
  -Xbootclasspath/a:target/agent/ttl-agent.jar \
  -jar target/your-app.jar
```

## 4. Docker 部署配置

在生产环境下，可以通过以下 Dockerfile 模板进行构建。它确保了 Agent 文件被正确复制到镜像中，并在启动命令中生效。

```dockerfile
# 使用 JDK 17 作为基础镜像
FROM openjdk:17-jdk

# 1. 设置时区
RUN ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime

# 2. 指定工作目录
WORKDIR /home/app

# 3. 复制文件
# 复制构建好的 Agent 脚本
COPY ./target/agent/ttl-agent.jar /home/app/ttl-agent.jar
# 复制应用 Jar 包 (请根据实际文件名修改)
COPY ./target/tail-assigment-bootstrap-1.0.0.jar /home/app/tail-assigment-bootstrap.jar

# 4. 声明端口
EXPOSE 8099

# 5. 执行命令启动
ENTRYPOINT ["java", \
            "-javaagent:/home/app/ttl-agent.jar", \
            "-Xbootclasspath/a:/home/app/ttl-agent.jar", \
            "-jar", \
            "tail-assigment-bootstrap.jar", \
            "--spring.profiles.active=pre"]
```

---

### 关键参数说明
- `-javaagent`: 指定 TTL Agent 的路径，用于在类加载时增强线程池相关的类。
- `-Xbootclasspath/a`: 将 Agent 添加到启动类路径，确保基础类加载器能找到 TTL 相关类。