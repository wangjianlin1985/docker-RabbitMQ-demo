# RabbitMQ 学习 Demo

> 使用 **Docker** 部署 RabbitMQ，并通过 **Spring AMQP（Spring Boot）** 逐个演示 RabbitMQ 的常见消息模型：HelloWorld、Work Queue、Fanout、Direct、Topic，以及 JSON 对象消息转换。

![Java](https://img.shields.io/badge/Java-1.8-orange)
![Spring Boot](https://img.shields.io/badge/Spring%20Boot-2.3.9-brightgreen)
![RabbitMQ](https://img.shields.io/badge/RabbitMQ-3--management-ff6600)
![Docker](https://img.shields.io/badge/Docker-ready-2496ED)

## 目录

- [项目简介](#项目简介)
- [技术栈](#技术栈)
- [项目结构](#项目结构)
- [快速开始](#快速开始)
- [消息模型演示](#消息模型演示)
- [对象消息（JSON 转换）](#对象消息json-转换)
- [常见问题](#常见问题)
- [声明](#声明)

## 项目简介

项目是一个 Maven 多模块工程，包含两个模块：

| 模块 | 说明 |
| --- | --- |
| `publisher` | 消息发布者：用单元测试的方式发送各种类型的消息 |
| `consumer` | 消息消费者：Spring Boot 应用，声明交换机 / 队列并监听消息 |

两个模块都分别提供了两套写法，方便对比学习：

- **原生 Java 客户端**（`helloworld` 包）：直接使用 `com.rabbitmq.client` 手动创建连接、通道、队列并收发消息
- **Spring AMQP**（`spring` 包 / `listener` 包）：使用 `RabbitTemplate` 发送，`@RabbitListener` 监听

## 技术栈

| 类别 | 技术 |
| --- | --- |
| 基础框架 | Spring Boot 2.3.9.RELEASE、Java 8、Maven |
| 消息队列 | RabbitMQ 3（`rabbitmq:3-management` 镜像）、`spring-boot-starter-amqp` |
| 消息转换 | Jackson（`Jackson2JsonMessageConverter`） |
| 测试 | JUnit 4 + Spring Boot Test |
| 部署 | Docker |

## 项目结构

```
.
├── 创建RabbitMQ容器命令.txt
└── mq-demo
    ├── pom.xml                                   # 父工程
    ├── publisher                                 # 消息发布者
    │   ├── src/main
    │   │   ├── java/cn/itcast/mq/PublisherApplication.java   # 启动类（配置 JSON 消息转换器）
    │   │   └── resources/application.yml                      # RabbitMQ 连接配置
    │   └── src/test/java/cn/itcast/mq
    │       ├── helloworld/PublisherTest.java     # 原生客户端发送消息
    │       └── spring/SpringAmqpTest.java        # Spring AMQP 发送各类消息
    └── consumer                                  # 消息消费者
        ├── src/main
        │   ├── java/cn/itcast/mq
        │   │   ├── ConsumerApplication.java      # 启动类（配置 JSON 消息转换器）
        │   │   ├── config/FanoutConfig.java      # 声明 Fanout 交换机、队列、绑定关系
        │   │   └── listener/SpringRabbitListener.java   # 所有队列的监听器
        │   └── resources/application.yml         # RabbitMQ 连接配置（含 prefetch）
        └── src/test/java/cn/itcast/mq/helloworld/ConsumerTest.java   # 原生客户端接收消息
```

## 快速开始

### 1. 环境要求

- JDK 1.8
- Maven 3.6+
- Docker
- IntelliJ IDEA（推荐，用于直接运行测试方法和启动类）

### 2. 用 Docker 部署 RabbitMQ

```bash
docker run \
  -e RABBITMQ_DEFAULT_USER=itcast \
  -e RABBITMQ_DEFAULT_PASS=123321 \
  --name rabbitmq \
  --hostname mq1 \
  -p 15672:15672 \
  -p 5672:5672 \
  -d \
  rabbitmq:3-management
```

| 端口 | 用途 |
| --- | --- |
| `5672` | AMQP 协议端口（应用连接用） |
| `15672` | Web 管理控制台 |

启动后访问管理控制台：<http://localhost:15672>，账号 `itcast`，密码 `123321`。

### 3. 修改连接配置

项目中的 RabbitMQ 地址是作者本机的 `192.168.1.3`，请改成你自己的地址（本机 Docker 一般是 `127.0.0.1` 或 `localhost`）。共有 **4 处**：

| 文件 | 位置 |
| --- | --- |
| `publisher/src/main/resources/application.yml` | `spring.rabbitmq.host` |
| `consumer/src/main/resources/application.yml` | `spring.rabbitmq.host` |
| `publisher/src/test/java/.../helloworld/PublisherTest.java` | `factory.setHost("192.168.1.3")` |
| `consumer/src/test/java/.../helloworld/ConsumerTest.java` | `factory.setHost("192.168.1.3")` |

```yaml
spring:
  rabbitmq:
    host: 127.0.0.1      # RabbitMQ 的 IP 地址
    port: 5672
    username: itcast
    password: 123321
    virtual-host: /
```

> 🔐 账号密码是教学用的示例值，请勿在真实环境中使用。

### 4. 创建 `simple.queue`

`SpringRabbitListener` 监听了 `simple.queue`，但这个队列**不是由 Spring 代码声明的**，启动消费者之前它必须已经存在。二选一：

- **方式 A（推荐）**：运行原生客户端的 `PublisherTest#testSendMessage`（或 `ConsumerTest` 的 `main` 方法），它们会用 `queueDeclare` 创建 `simple.queue`
- **方式 B**：在管理控制台 Queues 页面手动添加名为 `simple.queue` 的队列

> ⚠️ 两种方式不要混用：原生客户端声明的是**非持久化**队列，如果你在控制台手动创建成持久化队列，再运行原生客户端会因为属性不一致而报 `PRECONDITION_FAILED`。

### 5. 启动消费者

运行 `consumer` 模块的 `ConsumerApplication`。启动时会自动创建：

- `itcast.fanout` 交换机、`fanout.queue1`、`fanout.queue2`（来自 `FanoutConfig`）
- `itcast.direct` 交换机、`direct.queue1`、`direct.queue2`（来自 `@QueueBinding`）
- `itcast.topic` 交换机、`topic.queue1`、`topic.queue2`（来自 `@QueueBinding`）
- `object.queue`（来自 `FanoutConfig`）

> 💡 **一定要先启动消费者，再发消息。** 交换机和队列都由消费者创建，否则消息发到不存在的交换机会丢失 / 报错。

### 6. 运行发布者测试

在 IDEA 中打开 `publisher/src/test/java/cn/itcast/mq/spring/SpringAmqpTest.java`，逐个运行测试方法，在消费者控制台观察输出。也可以用命令行（需先 `cd mq-demo`）：

```bash
mvn test -pl publisher -Dtest=SpringAmqpTest#testSendFanoutExchange
```

> 项目没有配置 `spring-boot-maven-plugin`，因此消费者建议直接在 IDEA 中运行 `ConsumerApplication`。

## 消息模型演示

### 1. HelloWorld（基本队列）

```
publisher ──▶ simple.queue ──▶ consumer
```

- 发送：`PublisherTest#testSendMessage`（原生客户端）或 `SpringAmqpTest#testSendMessage2SimpleQueue`（Spring AMQP）
- 接收：`ConsumerTest#main`（原生客户端）或 `@RabbitListener(queues = "simple.queue")`

### 2. Work Queue（工作队列）

多个消费者监听同一个队列，共同处理消息，提高处理速度。

```
                      ┌──▶ 消费者1（处理快，sleep 20ms）
publisher ──▶ simple.queue
                      └──▶ 消费者2（处理慢，sleep 200ms）
```

- 发送：`SpringAmqpTest#testSendMessage2WorkQueue`，循环发送 50 条消息
- 接收：`listenWorkQueue1`、`listenWorkQueue2` 监听同一个 `simple.queue`
- 关键配置：消费者 `application.yml` 中的 `spring.rabbitmq.listener.simple.prefetch: 1`，表示每次只预取 1 条消息，处理完才取下一条，实现 **“能者多劳”**：处理快的消费者会处理更多消息。去掉该配置时，消息会被平均分配，与消费者处理速度无关。

### 3. Fanout（广播）

交换机把消息发给**所有**绑定的队列，忽略 RoutingKey。

```
                       ┌──▶ fanout.queue1 ──▶ 消费者
publisher ──▶ itcast.fanout
                       └──▶ fanout.queue2 ──▶ 消费者
```

- 声明：`FanoutConfig`（`FanoutExchange` + `Queue` + `Binding` 三个 Bean）
- 发送：`SpringAmqpTest#testSendFanoutExchange`
- 预期：`fanout.queue1` 和 `fanout.queue2` **都**收到消息

### 4. Direct（定向）

交换机根据消息的 RoutingKey，把消息发给**绑定了相同 Key** 的队列。

```
                                  ┌─ key: red, blue ──▶ direct.queue1
publisher ──(RoutingKey)──▶ itcast.direct
                                  └─ key: red, yellow ─▶ direct.queue2
```

- 声明：`@RabbitListener` + `@QueueBinding` 注解（队列、交换机、Key 一次性声明）
- 发送：`SpringAmqpTest#testSendDirectExchange`（RoutingKey = `red`）
- 预期：

| RoutingKey | direct.queue1 | direct.queue2 |
| --- | --- | --- |
| `red` | ✅ | ✅ |
| `blue` | ✅ | ❌ |
| `yellow` | ❌ | ✅ |

> 测试方法默认发送 `red`，要验证其他情况修改 `convertAndSend` 的第二个参数即可。

### 5. Topic（主题）

与 Direct 类似，但 RoutingKey 是用 `.` 分隔的多个单词，绑定 Key 支持通配符：

- `#`：匹配 0 个或多个单词
- `*`：匹配 1 个单词

```
                                 ┌─ binding key: china.#  ──▶ topic.queue1
publisher ──(RoutingKey)──▶ itcast.topic
                                 └─ binding key: #.news   ──▶ topic.queue2
```

- 发送：`SpringAmqpTest#testSendTopicExchange`（RoutingKey = `china.weather`）
- 预期：

| RoutingKey | topic.queue1（`china.#`） | topic.queue2（`#.news`） |
| --- | --- | --- |
| `china.weather` | ✅ | ❌ |
| `china.news` | ✅ | ✅ |
| `japan.news` | ❌ | ✅ |

## 对象消息（JSON 转换）

Spring AMQP 默认使用 JDK 序列化，体积大、有安全风险、可读性差。本项目在 **发布者和消费者的启动类** 中都注册了 `Jackson2JsonMessageConverter`，消息会以 JSON 格式传输：

```java
@Bean
public MessageConverter messageConverter(){
    return new Jackson2JsonMessageConverter();
}
```

- 消费者监听：`@RabbitListener(queues = "object.queue")`，参数类型为 `Map<String, Object>`
- 队列声明：`FanoutConfig` 中的 `objectQueue` Bean

仓库中的 `SpringAmqpTest` 没有对应的发送方法，可以自行添加一个来验证：

```java
@Test
public void testSendObjectQueue() {
    Map<String, Object> msg = new HashMap<>();
    msg.put("name", "柳岩");
    msg.put("age", 21);
    rabbitTemplate.convertAndSend("object.queue", msg);
}
```

## 常见问题

**Q：启动消费者报 `NOT_FOUND - no queue 'simple.queue'`？**
`simple.queue` 需要提前创建，见 [快速开始 - 第 4 步](#4-创建-simplequeue)。

**Q：连接超时 / `Connection refused`？**
确认容器已启动（`docker ps`）、配置里的 IP 已改成自己的、`5672` 端口没被防火墙拦截。

**Q：认证失败 `ACCESS_REFUSED`？**
账号密码需要与 `docker run` 时的 `RABBITMQ_DEFAULT_USER` / `RABBITMQ_DEFAULT_PASS` 一致（`itcast` / `123321`）。

**Q：发送了消息，消费者收不到？**
- 是否先启动了消费者（交换机、队列由消费者创建）
- 在管理控制台 Exchanges / Queues 页面确认交换机、队列和绑定关系是否存在
- 检查发送的交换机名称和 RoutingKey 是否与绑定一致

**Q：重启 RabbitMQ 容器后，队列还在吗？**
容器没有挂载数据卷。只是 `docker restart` 的话，持久化的队列和交换机还在；如果删除容器重新 `docker run`，数据会丢失，需要重新创建（重新启动消费者即可）。

**Q：Work Queue 为什么两个消费者的消息数量差不多？**
检查消费者的 `application.yml` 是否配置了 `listener.simple.prefetch: 1`，没有的话默认会先把消息平均分发。

## 声明

本项目为 RabbitMQ / Spring AMQP 的**学习与交流**用途，请勿用于商业用途。
