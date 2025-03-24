+++
title = "简易代码沙箱执行服务"
date = "2025-03-24"
tags = ["Tech"]
summary = "简单的沙箱实现"
categories = ["Tech"]
+++

在开发在线评测系统、教学平台或 AI 编程辅助平台时，经常需要**运行用户提交的代码**。为了保证安全性与资源隔离，通常会使用 **Docker 容器沙箱** 来执行这些代码。

本文将一步步搭建一个基于 Spring Boot 的代码执行服务：

- 使用 Docker 容器执行 Java/Python 代码
- 对容器做 CPU、内存限制，防止滥用资源
- 使用 Spring Boot 提供 REST 接口
- 使用 JDK 21（兼容新特性）

## 项目结构

```text
sandbox-runner/
├── Dockerfile                  # 构建沙箱镜像（含 JDK 21 + Python3）
├── run-java.sh                 # Java 执行脚本
├── run-python.sh               # Python 执行脚本
├── src/main/java/com/example/sandbox/
│   ├── SandboxController.java  # 提供 REST 接口
│   ├── SandboxService.java     # 执行 docker 命令
│   └── SandboxRunnerApplication.java  # Spring Boot 启动类
├── src/main/resources/application.yml
└── pom.xml
```

### 🧭 整体执行流程图

![整体图](/images/3.png)

## Docker 镜像

用于运行用户代码的容器镜像：

```dockerfile
FROM eclipse-temurin:21-jdk-alpine

RUN apk add --no-cache bash python3 py3-pip

WORKDIR /app

COPY run-java.sh /usr/local/bin/run-java
COPY run-python.sh /usr/local/bin/run-python

RUN chmod +x /usr/local/bin/run-java /usr/local/bin/run-python
```

### Java 执行脚本：`run-java.sh`

```bash
#!/bin/bash
echo "$1" > Main.java
timeout 5 javac Main.java 2> compile_err.txt
if [ $? -ne 0 ]; then
  echo "--- Compilation Failed ---"
  cat compile_err.txt
  exit 1
fi
timeout 5 java Main
```

### Python 执行脚本：`run-python.sh`

```bash
#!/bin/bash
echo "$1" > main.py
timeout 5 python3 main.py
```

## Spring Boot 项目核心

### `SandboxRunnerApplication.java`

```java
@SpringBootApplication
public class SandboxRunnerApplication {
    public static void main(String[] args) {
        SpringApplication.run(SandboxRunnerApplication.class, args);
    }
}
```

### `SandboxController.java`

```java
@RestController
@RequestMapping("/run")
public class SandboxController {
    private final SandboxService sandboxService = new SandboxService();

    @PostMapping("/{lang}")
    public ResponseEntity<String> runCode(@PathVariable("lang") String lang, @RequestBody String code) {
        String output = sandboxService.runInDocker(lang, code);
        return ResponseEntity.ok(output);
    }
}
```

### `SandboxService.java`

```java
public class SandboxService {
    public String runInDocker(String lang, String code) {
        String image = "sandbox-runner:latest";
        String cmd = switch (lang) {
            case "java" -> "run-java";
            case "python" -> "run-python";
            default -> throw new IllegalArgumentException("Unsupported lang");
        };

        try {
            Path temp = Files.createTempFile(lang + "_code", ".txt");
            Files.writeString(temp, code);

            ProcessBuilder pb = new ProcessBuilder(
                "docker", "run", "--rm", "--cpus=0.5", "--memory=256m",
                "-v", temp.toAbsolutePath() + ":/tmp/code.txt",
                image, cmd, "$(cat /tmp/code.txt)"
            );

            pb.redirectErrorStream(true);
            Process p = pb.start();
            return new String(p.getInputStream().readAllBytes());
        } catch (IOException e) {
            return "Execution error: " + e.getMessage();
        }
    }
}
```

### 🔁 系统调用流程图

![系统调用流程图](/images/1.png)

## Maven 配置

```xml
<properties>
    <java.version>21</java.version>
</properties>

<build>
  <plugins>
    <plugin>
      <groupId>org.apache.maven.plugins</groupId>
      <artifactId>maven-compiler-plugin</artifactId>
      <version>3.11.0</version>
      <configuration>
        <source>21</source>
        <target>21</target>
        <compilerArgs>
          <arg>-parameters</arg>
        </compilerArgs>
      </configuration>
    </plugin>
  </plugins>
</build>
```

## 常见问题

### 1. 编译失败：不支持 Java 5

```text
不再支持源选项 5。请使用 8 或更高版本。
```

**解决：** pom.xml 添加 `<source>21</source>` 和 `<target>21</target>`

### 2. 缺少 main 方法无法启动

添加 `SandboxRunnerApplication.java` 并标注 `@SpringBootApplication`

### 3. 参数名缺失异常

```text
parameter name information not available via reflection
```

**解决：**

- 显式写参数名：`@PathVariable("lang")`
- 或在 Maven 添加：`<arg>-parameters</arg>`

## 总结

本项目是一个最小可运行的“沙箱代码运行平台”原型，具备如下优势：

- ✅ 多语言执行（可扩展）
- ✅ Docker 隔离，资源可控
- ✅ 安全性高，不接触宿主系统
- ✅ 可集成 K8s、Ingress、限流、日志收集等功能

### ✅ 总体架构回顾图

![](/images/2.png)


欢迎参考并扩展！🚀