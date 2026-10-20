# Docker + RabbitMQ 学习案例

这是一个基于 **Java 8、Maven、Spring Boot、Spring AMQP 和 RabbitMQ** 的消息队列学习项目。项目拆分为消息生产者（`publisher`）和消息消费者（`consumer`），通过测试代码和监听器演示 RabbitMQ 的常见消息模型。

> 说明：当前项目压缩包中没有提供 `Dockerfile` 或 `docker-compose.yml`。下面使用 Docker 官方 RabbitMQ 镜像启动 RabbitMQ 服务，Java 项目在宿主机运行并连接容器映射出来的端口。

## 功能与知识点

- **Hello World / 简单队列**：使用 RabbitMQ Java Client 创建队列、发送和接收消息。
- **Work Queue / 工作队列**：多个消费者监听同一个队列，观察消息分发与消费速度差异。
- **Spring AMQP**：使用 `RabbitTemplate` 发送消息，使用 `@RabbitListener` 接收消息。
- **Fanout Exchange / 广播交换机**：一条消息路由到绑定的多个队列。
- **Direct Exchange / 直连交换机**：根据精确匹配的 Routing Key 路由消息。
- **Topic Exchange / 主题交换机**：使用 `*`、`#` 等通配符模式匹配 Routing Key。
- **对象消息队列**：项目中包含 `object.queue` 的 Map 消息监听示例。

## 项目结构

```text
mq-demo/
├── pom.xml                         # Maven 父工程及公共依赖
├── publisher/                      # 消息生产者
│   ├── pom.xml
│   └── src/
│       ├── main/
│       │   ├── java/cn/itcast/mq/
│       │   │   └── PublisherApplication.java
│       │   └── resources/application.yml
│       └── test/java/cn/itcast/mq/
│           ├── helloworld/PublisherTest.java
│           └── spring/SpringAmqpTest.java
└── consumer/                       # 消息消费者
    ├── pom.xml
    └── src/
        ├── main/
        │   ├── java/cn/itcast/mq/
        │   │   ├── ConsumerApplication.java
        │   │   ├── config/FanoutConfig.java
        │   │   └── listener/SpringRabbitListener.java
        │   └── resources/application.yml
        └── test/java/cn/itcast/mq/helloworld/ConsumerTest.java
```

## 环境要求

- Docker（用于运行 RabbitMQ）
- JDK 8
- Maven 3.x
- IntelliJ IDEA 或其他 Java IDE（可选）

项目使用 Spring Boot `2.3.9.RELEASE`，编译目标为 Java 8。

## 1. 使用 Docker 启动 RabbitMQ

先拉取并启动带 Web 管理界面的 RabbitMQ 镜像：

```bash
docker run -d \
  --name rabbitmq \
  --hostname rabbitmq \
  -p 5672:5672 \
  -p 15672:15672 \
  -e RABBITMQ_DEFAULT_USER=itcast \
  -e RABBITMQ_DEFAULT_PASS=123321 \
  rabbitmq:3-management
```

Windows PowerShell 可以使用单行命令：

```powershell
docker run -d --name rabbitmq --hostname rabbitmq -p 5672:5672 -p 15672:15672 -e RABBITMQ_DEFAULT_USER=itcast -e RABBITMQ_DEFAULT_PASS=123321 rabbitmq:3-management
```

查看容器状态和日志：

```bash
docker ps
docker logs -f rabbitmq
```

浏览器打开管理页面：

- 地址：<http://localhost:15672>
- 用户名：`itcast`
- 密码：`123321`

首次启动时请等待 RabbitMQ 完成初始化，再运行 Java 项目。

> **安全提示：** 上述账号密码仅用于本地学习。不要将示例密码用于公网服务或生产环境。

## 2. 配置 RabbitMQ 连接

生产者配置文件：

`publisher/src/main/resources/application.yml`

消费者配置文件：

`consumer/src/main/resources/application.yml`

当前压缩包中的配置默认连接 `192.168.1.3:5672`。如果 RabbitMQ 就运行在本机 Docker 容器中，请将两个文件中的连接地址改为 `localhost`，并确认账号密码与启动容器时设置的一致：

```yaml
spring:
  rabbitmq:
    host: localhost
    port: 5672
    username: itcast
    password: 123321
    virtual-host: /
```

消费者配置中还设置了：

```yaml
spring:
  rabbitmq:
    listener:
      simple:
        prefetch: 1
```

`prefetch: 1` 表示消费者在确认当前消息前，最多预取一条未确认消息，有助于观察多个消费者之间的任务分配。

## 3. 编译项目

在 `mq-demo` 根目录执行：

```bash
mvn clean package
```

如果只想跳过测试、先检查项目能否打包：

```bash
mvn clean package -DskipTests
```

## 4. 启动消费者

先启动消费者，使队列监听器准备好接收消息。可以在 IntelliJ IDEA 中运行：

`consumer/src/main/java/cn/itcast/mq/ConsumerApplication.java`

消费者启动后，`SpringRabbitListener` 中声明的监听器会监听相应队列；Fanout 队列及绑定由 `FanoutConfig` 配置类声明，Direct 和 Topic 相关队列、交换机与绑定由 `@RabbitListener` 注解声明。

## 5. 发送测试消息

在 IntelliJ IDEA 中打开 `publisher/src/test/java` 下的测试类，按需单独运行测试方法。不要把所有测试方法当成一个必须连续执行的流程。

| 测试方法 | 用途 | 关键配置 |
|---|---|---|
| `PublisherTest.testSendMessage()` | 使用 RabbitMQ 原生 Java Client 向简单队列发送消息 | `simple.queue` |
| `SpringAmqpTest.testSendMessage2SimpleQueue()` | 使用 `RabbitTemplate` 发送简单队列消息 | `simple.queue` |
| `SpringAmqpTest.testSendMessage2WorkQueue()` | 连续发送 50 条消息，观察工作队列消费 | `simple.queue` |
| `SpringAmqpTest.testSendFanoutExchange()` | 发送广播消息 | 交换机 `itcast.fanout` |
| `SpringAmqpTest.testSendDirectExchange()` | 使用 `red` 路由键发送消息 | 交换机 `itcast.direct` |
| `SpringAmqpTest.testSendTopicExchange()` | 使用 `china.weather` 路由键发送消息 | 交换机 `itcast.topic` |

### 路由规则示例

- **Fanout**：`itcast.fanout` 将消息广播到绑定的 `fanout.queue1` 和 `fanout.queue2`。
- **Direct**：
  - `direct.queue1` 绑定路由键 `red`、`blue`。
  - `direct.queue2` 绑定路由键 `red`、`yellow`。
  - 因此测试消息使用 `red` 时，两个队列都能匹配。
- **Topic**：
  - `topic.queue1` 绑定 `china.#`，可匹配 `china.weather`。
  - `topic.queue2` 绑定 `#.news`，可匹配以 `news` 结尾的主题，例如 `china.news`。
  - 因此 `china.weather` 会匹配 `topic.queue1`，不会匹配 `topic.queue2`。

## 6. 查看运行结果

- 在消费者控制台查看监听器打印的消息。
- 打开 RabbitMQ 管理页面，在 **Queues and Streams** 中查看队列，在 **Exchanges** 中查看交换机及绑定。
- 对于工作队列，可以观察两个消费者打印的消息顺序和时间差。由于示例中两个监听方法的处理耗时不同，消息分配与完成处理的速度可能不同。

## 常见问题

### 连接 RabbitMQ 失败

1. 确认容器已启动：`docker ps`
2. 确认端口映射包含 `5672:5672`。
3. 确认 `application.yml` 中的主机地址、端口、用户名、密码和虚拟主机正确。
4. 如果 Java 程序运行在另一台机器或容器中，`localhost` 指向的是程序所在环境，而不是 RabbitMQ 所在主机；请改成该环境能够访问的 RabbitMQ 地址。

### 管理页面打不开

确认容器映射了 `15672:15672`，并等待服务启动完成。管理页面使用 `15672`，Java AMQP 连接使用 `5672`，两者不要混淆。

### 测试类连接失败

部分测试类会直接把 RabbitMQ 地址和账号密码写在 Java 代码中（例如 `PublisherTest`、`ConsumerTest`），并不会读取 `application.yml`。如果运行这些原生客户端示例，请同步检查对应测试代码中的连接参数。

### 消息没有被消费

确认消费者应用已启动，并检查监听器声明的队列名称、交换机名称和 Routing Key 是否一致。Direct 和 Topic 模式下，Routing Key 不匹配时，消息不会路由到相应队列。

## 停止与清理 Docker 容器

停止 RabbitMQ：

```bash
docker stop rabbitmq
```

再次启动：

```bash
docker start rabbitmq
```

删除容器（会删除容器内未持久化的数据）：

```bash
docker rm -f rabbitmq
```

## 学习建议

建议按以下顺序学习：

1. 先理解队列、生产者、消费者和消息的基本概念。
2. 运行原生 Java Client 的简单队列示例。
3. 学习 Spring AMQP 的 `RabbitTemplate` 和 `@RabbitListener`。
4. 对比 Work Queue 的多个消费者行为。
5. 分别修改 Routing Key，观察 Fanout、Direct、Topic 三种交换机的路由差异。
6. 尝试进一步学习消息确认、持久化、死信队列、延迟消息、重试机制和生产者确认。

## 免责声明

本仓库是用于学习与实验的示例项目。请在本地测试环境中运行，并根据自己的 RabbitMQ 地址和账号调整配置。项目中示例凭据仅用于演示，请勿直接用于生产环境。
