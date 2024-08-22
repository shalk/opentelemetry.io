---
title: 开始使用
description: 在不到5分钟的时间内为您的应用程序获取遥测数据！
weight: 10
default_lang_commit: 5d26b8a551
---

<!-- markdownlint-disable blanks-around-fences -->
<?code-excerpt path-base="examples/java/getting-started"?>

本页将向您展示如何在 Java 中开始使用 Opentelemetry。

你将学会如何给一个简单的 Java 应用程序自动插桩，通过这种方式[追踪][]、[指标][]和[日志][]会发送到控制台。

## 准备

确保你本地安装了如下这些:

- Java JDK 17+, 由于使用了 Spring Boot 3; [否则需要 Java 8+][java-vers]
- [Gradle](https://gradle.org/)

## 示例应用

以下示例使用了一个基本的 [Spring Boot] 应用程序。你可以使用其他的 Web 框架，例如 Apache Wicket 或者 Play。有关库和受支持框架的完整列表, 请查阅[注册表](/ecosystem/registry/?component=instrumentation&language=java).

有关更详细的示例, 请查阅[示例](/docs/languages/java/examples/).

### 依赖

首先, 在名称为 `java-simple` 的新目录中准备环境。在该目录中，创建一个名为 `build.gradle.kts` 的文件，内容如下：

```kotlin
plugins {
  id("java")
  id("org.springframework.boot") version "3.0.6"
  id("io.spring.dependency-management") version "1.1.0"
}

sourceSets {
  main {
    java.setSrcDirs(setOf("."))
  }
}

repositories {
  mavenCentral()
}

dependencies {
  implementation("org.springframework.boot:spring-boot-starter-web")
}
```

### 创建并启动 HTTP 服务器

在同一文件夹中，创建一个名为的文件 `DiceApplication.java` 并将以下代码添加到该文件：

<!-- prettier-ignore-start -->
<?code-excerpt "src/main/java/otel/DiceApplication.java"?>
```java
package otel;

import org.springframework.boot.Banner;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class DiceApplication {
  public static void main(String[] args) {
    SpringApplication app = new SpringApplication(DiceApplication.class);
    app.setBannerMode(Banner.Mode.OFF);
    app.run(args);
  }
}
```
<!-- prettier-ignore-end -->

创建另一个名为 `RollController.java` 的文件，并将以下代码添加到该文件中：

<!-- prettier-ignore-start -->
<?code-excerpt "src/main/java/otel/RollController.java"?>
```java
package otel;

import java.util.Optional;
import java.util.concurrent.ThreadLocalRandom;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class RollController {
  private static final Logger logger = LoggerFactory.getLogger(RollController.class);

  @GetMapping("/rolldice")
  public String index(@RequestParam("player") Optional<String> player) {
    int result = this.getRandomNumber(1, 6);
    if (player.isPresent()) {
      logger.info("{} is rolling the dice: {}", player.get(), result);
    } else {
      logger.info("Anonymous player is rolling the dice: {}", result);
    }
    return Integer.toString(result);
  }

  public int getRandomNumber(int min, int max) {
    return ThreadLocalRandom.current().nextInt(min, max + 1);
  }
}
```
<!-- prettier-ignore-end -->

使用以下命令构建并运行应用程序，然后在 Web 浏览器中打开
<http://localhost:8080/rolldice> 以确保其正常运行。

```sh
gradle assemble
java -jar ./build/libs/java-simple.jar
```

## 插桩

接下来, 你将使用 [Java agent](/docs/zero-code/java/agent/) 在应用启动时自动插桩。 虽然你可以通过多种方式 [配置 Java agent][]，但以下步骤将使用环境变量。

1. 从 `opentelemetry-java-instrumentation` 仓库的 [发行版][] 
   下载 [opentelemetry-javaagent.jar][]. JAR 文件包含 agent 和所有的自动插桩的包：

   ```console
   curl -L -O https://github.com/open-telemetry/opentelemetry-java-instrumentation/releases/latest/download/opentelemetry-javaagent.jar
   ```

   {{% alert color="info" %}}<i class="fas fa-edit"></i> 记下文件的路径。{{% /alert %}}

2. 设置并导出变量用于指定 Java agent JAR 和 一个[控制台导出器][]，
   使用适合你 shell 或终端环境的符号 &mdash; 我们用类 bash 的 shell 符号举例：

   ```sh
   export JAVA_TOOL_OPTIONS="-javaagent:PATH/TO/opentelemetry-javaagent.jar" \
     OTEL_TRACES_EXPORTER=logging \
     OTEL_METRICS_EXPORTER=logging \
     OTEL_LOGS_EXPORTER=logging \
     OTEL_METRIC_EXPORT_INTERVAL=15000
   ```

   {{% alert title="Important" color="warning" %}}

   - 用你的 JAR 的路径替换上面的 `PATH/TO`。
   - 如我们上面所示，**仅在测试期间** 设置 `OTEL_METRIC_EXPORT_INTERVAL` 低于默认值, 
     可以帮你快速确定指标已经生成。
     

   {{% /alert %}}

3. 再次运行你的 **应用** ：

   ```console
   $ java -jar ./build/libs/java-simple.jar
   ...
   ```

   Note the output from the `otel.javaagent`.

4. From _another_ terminal, send a request using `curl`:

   ```sh
   curl localhost:8080/rolldice
   ```

5. Stop the server process.

At step 4, you should have seen trace & log output from the server and client
that looks something like this (trace output is line-wrapped for convenience):

```sh
[otel.javaagent 2023-04-24 17:33:54:567 +0200] [http-nio-8080-exec-1] INFO
io.opentelemetry.exporter.logging.LoggingSpanExporter - 'RollController.index' :
 70c2f04ec863a956e9af975ba0d983ee 7fd145f5cda13625 INTERNAL [tracer:
 io.opentelemetry.spring-webmvc-6.0:1.25.0-alpha] AttributesMap{data=
 {thread.id=39, thread.name=http-nio-8080-exec-1}, capacity=128,
 totalAddedValues=2}
[otel.javaagent 2023-04-24 17:33:54:568 +0200] [http-nio-8080-exec-1] INFO
io.opentelemetry.exporter.logging.LoggingSpanExporter - 'GET /rolldice' :
70c2f04ec863a956e9af975ba0d983ee 647ad186ad53eccf SERVER [tracer:
io.opentelemetry.tomcat-10.0:1.25.0-alpha] AttributesMap{
  data={user_agent.original=curl/7.87.0, net.host.name=localhost,
  net.transport=ip_tcp, http.target=/rolldice, net.sock.peer.addr=127.0.0.1,
  thread.name=http-nio-8080-exec-1, net.sock.peer.port=53422,
  http.route=/rolldice, net.sock.host.addr=127.0.0.1, thread.id=39,
  net.protocol.name=http, http.status_code=200, http.scheme=http,
  net.protocol.version=1.1, http.response_content_length=1,
  net.host.port=8080, http.method=GET}, capacity=128, totalAddedValues=17}
```

At step 5, when stopping the server, you should see an output of all the metrics
collected (metrics output is line-wrapped and shortened for convenience):

```sh
[otel.javaagent 2023-04-24 17:34:25:347 +0200] [PeriodicMetricReader-1] INFO
io.opentelemetry.exporter.logging.LoggingMetricExporter - Received a collection
 of 19 metrics for export.
[otel.javaagent 2023-04-24 17:34:25:347 +0200] [PeriodicMetricReader-1] INFO
io.opentelemetry.exporter.logging.LoggingMetricExporter - metric:
ImmutableMetricData{resource=Resource{schemaUrl=
https://opentelemetry.io/schemas/1.19.0, attributes={host.arch="aarch64",
host.name="OPENTELEMETRY", os.description="Mac OS X 13.3.1", os.type="darwin",
process.command_args=[/bin/java, -jar, java-simple.jar],
process.executable.path="/bin/java", process.pid=64497,
process.runtime.description="Homebrew OpenJDK 64-Bit Server VM 20",
process.runtime.name="OpenJDK Runtime Environment",
process.runtime.version="20", service.name="java-simple",
telemetry.auto.version="1.25.0", telemetry.sdk.language="java",
telemetry.sdk.name="opentelemetry", telemetry.sdk.version="1.25.0"}},
instrumentationScopeInfo=InstrumentationScopeInfo{name=io.opentelemetry.runtime-metrics,
version=1.25.0, schemaUrl=null, attributes={}},
name=process.runtime.jvm.buffer.limit, description=Total capacity of the buffers
in this pool, unit=By, type=LONG_SUM, data=ImmutableSumData{points=
[ImmutableLongPointData{startEpochNanos=1682350405319221000,
epochNanos=1682350465326752000, attributes=
{pool="mapped - 'non-volatile memory'"}, value=0, exemplars=[]},
ImmutableLongPointData{startEpochNanos=1682350405319221000,
epochNanos=1682350465326752000, attributes={pool="mapped"},
value=0, exemplars=[]},
ImmutableLongPointData{startEpochNanos=1682350405319221000,
epochNanos=1682350465326752000, attributes={pool="direct"},
value=8192, exemplars=[]}], monotonic=false, aggregationTemporality=CUMULATIVE}}
...
```

## What next?

For more:

- Run this example with another [exporter][] for telemetry data.
- Try [zero-code instrumentation](/docs/zero-code/java/agent/) on one of your
  own apps.
- For light-weight customized telemetry, try [annotations][].
- Learn about [manual instrumentation][] and try out more [examples](/docs/languages/java/examples/).
- Take a look at the [OpenTelemetry Demo](/docs/demo/), which includes Java
  based [Ad Service](/docs/demo/services/ad/) and Kotlin based
  [Fraud Detection Service](/docs/demo/services/fraud-detection/)

[traces]: /docs/concepts/signals/traces/
[metrics]: /docs/concepts/signals/metrics/
[logs]: /docs/concepts/signals/logs/
[annotations]: /docs/zero-code/java/agent/annotations/
[configure the java agent]: /docs/zero-code/java/agent/configuration/
[console exporter]:
  https://github.com/open-telemetry/opentelemetry-java/blob/main/sdk-extensions/autoconfigure/README.md#logging-exporter
[exporter]:
  https://github.com/open-telemetry/opentelemetry-java/blob/main/sdk-extensions/autoconfigure/README.md#exporters
[java-vers]:
  https://github.com/open-telemetry/opentelemetry-java/blob/main/VERSIONING.md#language-version-compatibility
[manual instrumentation]: ../instrumentation
[opentelemetry-javaagent.jar]:
  https://github.com/open-telemetry/opentelemetry-java-instrumentation/releases/latest/download/opentelemetry-javaagent.jar
[发行版]:
  https://github.com/open-telemetry/opentelemetry-java-instrumentation/releases
[Spring Boot]: https://spring.io/guides/gs/spring-boot/
