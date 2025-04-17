# SpringBoot + Spring AI Alibaba 整合阿里云百炼DeepSeek大模型，小白也能轻松上手

Spring 官方开源了 Spring AI 框架，用来简化 Spring 开发者开发智能体应用的过程。随后阿里巴巴开源了 Spring AI Alibaba，它基于 Spring AI，同时与阿里云百炼大模型服务、通义系列大模型做了深度集成与最佳实践。基于 Spring AI Alibaba，Java 开发者可以非常方便地开发 AI 智能体应用。

![1740312879896-2e49c5fa-73ea-45c5-9589-3839c48e4f5e.webp](./img/tBCSlVg7zhOV6uHr/1740312879896-2e49c5fa-73ea-45c5-9589-3839c48e4f5e-645143.webp)

**开通阿里云百炼账号，获取API-KEY**

[官网卡顿怎么办？亲测有效！6种DeepSeek-R1满血模型替代方案分享](https://mp.weixin.qq.com/s?__biz=MzU1ODk1NTQ0Mg==&mid=2247485221&idx=1&sn=18f2cfc6b1fcf9ac39fa171e99ae964a&scene=21#wechat_redirect)

`<https://bailian.console.aliyun.com/#/model-market>`

其中 deepseek-r1 与 deepseek-v3 分别有 100  
万的免费 Token，部分蒸馏模型限时免费体验。

![1740312879960-d3cdc00c-522c-44de-bf9c-96a51378fed3.webp](./img/tBCSlVg7zhOV6uHr/1740312879960-d3cdc00c-522c-44de-bf9c-96a51378fed3-966814.webp)

**SpringBoot接入deepseek实战**

使用 Spring AI Alibaba 开发应用与使用普通 Spring Boot 没有什么区别，只需要增加 `spring-ai-alibaba-starter` 依赖，将 `ChatClient` Bean 注入就可以实现与模型聊天了。

注意：因为 Spring AI Alibaba 基于 Spring Boot 3.x 开发，因此本地 JDK 版本要求为 17 及以上。

1. **添加依赖**

首先，需要在项目中添加 `spring-ai-alibaba-starter` 依赖，它将通过 Spring Boot 自动装配机制初始化与阿里云通义大模型通信的 `ChatClient`、`ChatModel` 相关实例。

```plain

<dependency>
  <groupId>com.alibaba.cloud.ai</groupId>
  <artifactId>spring-ai-alibaba-starter</artifactId>
  <version>1.0.0-M5.1</version>
</dependency>
```

注意：由于 spring-ai 相关依赖包还没有发布到中央仓库，如出现 spring-ai-core 等相关依赖解析问题，请在您项目的 pom.xml 依赖中加入如下仓库配置。

```plain
<repositories>
  <repository>
    <id>spring-milestones</id>
    <name>Spring Milestones</name>
    <url>https://repo.spring.io/milestone</url>
    <snapshots>
      <enabled>false</enabled>
    </snapshots>
  </repository>
</repositories>
```

完整的pom文件如下

```plain
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.4.3</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>
    <groupId>com.fox</groupId>
    <artifactId>alibaba-ai-demo</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>alibaba-ai-demo</name>
    <description>alibaba-ai-demo</description>
    <url/>

    <properties>
        <java.version>17</java.version>
        <spring-ai.version>1.0.0-M5</spring-ai.version>
        <spring-ai-alibaba.version>1.0.0-M5.1</spring-ai-alibaba.version>
    </properties>

    <dependencies>

     <dependency>
            <groupId>com.alibaba.cloud.ai</groupId>
            <artifactId>spring-ai-alibaba-starter</artifactId>
            <version>${spring-ai-alibaba.version}</version>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.ai</groupId>
                <artifactId>spring-ai-bom</artifactId>
                <version>${spring-ai.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>


    <repositories>
        <!-- spring-ai 相关依赖包还没有发布到中央仓库-->
        <repository>
            <id>spring-milestones</id>
            <name>Spring Milestones</name>
            <url>https://repo.spring.io/milestone</url>
            <snapshots>
                <enabled>false</enabled>
            </snapshots>
        </repository>
    </repositories>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>

</project>
```

**2.配置 **`**application.yml**`****

指定 API-KEY（可通过访问阿里云百炼模型服务平台获取，有免费额度可用）

```plain
spring:
  application:
    name: alibaba-ai-demo

  ai:
    dashscope:
      api-key: ${AI_DASHSCOPE_API_KEY}   # api key
      chat:
        options:
          model: deepseek-r1   # 模型名称
```

**3.注入智能体代理 ChatClient**

接下来，在普通 Controller Bean 中注入 `ChatClient` 实例，这样你的 Bean 就具备与 AI 大模型智能对话的能力了。

```plain
@RestController
public class ChatController {

    private final ChatClient chatClient;

    public ChatController(ChatClient.Builder builder) {
        this.chatClient = builder.build();
    }

    @GetMapping("/chat")
    public String chat(@RequestParam(value = "input") String input) {
        return this.chatClient.prompt()
                .user(input)
                .call()
                .content();
    }
}
```

以上示例中，ChatClient 使用默认参数调用大模型，Spring AI Alibaba 还支持通过 `DashScopeChatOptions` 调整与模型对话时的参数，`DashScopeChatOptions` 支持两种不同维度的配置方式：

+ 全局默认值，即 `ChatClient` 实例初始化参数

可以在 `application.yml` 文件中指定

spring.ai.dashscope.chat.options.*

或调用构造函数

ChatClient.Builder.defaultOptions(options)完成配置初始化。

+ 每次 Prompt 调用前动态指定

```plain
String result = dashScopeChatClient
  .prompt(query)
  .options(DashScopeChatOptions.builder().withTopP(0.8).build())
  .call()
  .content();
```

4.启动服务后测试

![1740313095939-328fd2c1-3458-4957-8d85-c78bdea63d53.webp](./img/tBCSlVg7zhOV6uHr/1740313095939-328fd2c1-3458-4957-8d85-c78bdea63d53-191367.webp)

如果觉得这篇文章对你有所帮助，欢迎点个 **“在看”** 或分享给更多的小伙伴！更多技术干货，欢迎关注微信公众号【Fox爱分享】，解锁更多精彩内容！

![1740313095984-22912fd3-e15a-4948-8bfd-467d0dbfa39c.webp](./img/tBCSlVg7zhOV6uHr/1740313095984-22912fd3-e15a-4948-8bfd-467d0dbfa39c-016418.webp)

关注后回复"deepseek",获取deepseek实战资料

![1740313096032-5030eeb1-f3d8-4477-9253-f6b9aa5bca74.png](./img/tBCSlVg7zhOV6uHr/1740313096032-5030eeb1-f3d8-4477-9253-f6b9aa5bca74-697847.png)
