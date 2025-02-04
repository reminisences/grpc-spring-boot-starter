# 测试服务

[<- 返回索引](../index.md)

本节介绍如何为您的 grpc-service 编写测试用例。

如果你想要测试一个使用了 `@GrpcClient` 注解字段或一个 grpc 的 stub。 请参阅 [测试Grpc-Stubs](../client/testing.md)。

## 目录 <!-- omit in toc -->

- [前言](#前言)
- [测试服务](#测试服务)
- [有用的依赖项](#有用的依赖项)
- [单元测试](#单元测试)
  - [独立测试](#独立测试)
  - [基于Spring的测试](#基于Spring的测试)
- [集成测试](#集成测试)
- [gRPCurl](#grpcurl)

## 附加主题 <!-- omit in toc -->

- [入门指南](getting-started.md)
- [配置](configuration.md)
- [异常处理](exception-handling.md)
- [上下文数据 / Bean 的作用域](contextual-data.md)
- *测试服务*
- [服务端事件](events.md)
- [安全性](security.md)

## 前言

我们都知道测试对我们的应用程序是多么重要，所以我只会在这里向大家介绍几个链接：

- [Testing Spring](https://docs.spring.io/spring/docs/current/spring-framework-reference/testing.html)
- [Testing with JUnit](https://junit.org/junit5/docs/current/user-guide/#writing-tests)
- [grpc-spring-boot-starter's Tests](https://github.com/yidongnan/grpc-spring-boot-starter/tree/master/tests/src/test/java/net/devh/boot/grpc/test)

通常有三种方法来测试您的 grpc 服务：

- [直接测试](#unit-tests)
- [通过 grpc 测试](#integration-tests)
- [在生产环境中测试它们](#grpcurl) (构建时的自动化测试除外)

## 测试服务

让我们假设，我们希望测试以下服务：

````java
@GrpcService
public class MyServiceImpl extends MyServiceGrpc.MyServiceImplBase {

    private OtherDependency foobar;

    @Autowired
    public void setFoobar(OtherDependency foobar) {
        this.foobar = foobar;
    }

    @Override
    public void sayHello(HelloRequest request, StreamObserver<HelloReply> responseObserver) {
        HelloReply response = HelloReply.newBuilder()
                .setMessage("Hello ==> " + request.getName())
                .setCounter(foobar.getCount())
                .build();
        responseObserver.onNext(response);
        responseObserver.onComplete();
    }

}
````

## 有用的依赖项

在您开始编写自己的测试框架之前，您可能想要使用以下库来使您的工作更加简单。

> **注意：** Spring-Boot-Test已经包含一些依赖项，所以请确保您排除掉了冲突的版本。

对于Maven来说，添加以下依赖：

````xml
<!-- JUnit-Test-Framework -->
<dependency>
    <groupId>org.junit.jupiter</groupId>
    <artifactId>junit-jupiter-api</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.junit.jupiter</groupId>
    <artifactId>junit-jupiter-engine</artifactId>
    <scope>test</scope>
</dependency>
<!-- Grpc-Test-Support -->
<dependency>
    <groupId>io.grpc</groupId>
    <artifactId>grpc-testing</artifactId>
    <scope>test</scope>
</dependency>
<!-- Spring-Test-Support (Optional) -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
</dependency>
````

Gradle 使用：

````groovy
// JUnit-Test-Framework
testImplementation("org.junit.jupiter:junit-jupiter-api")
testRuntimeOnly("org.junit.jupiter:junit-jupiter-engine")
// Grpc-Test-Support
testImplementation("io.grpc:grpc-testing")
// Spring-Test-Support (Optional)
testImplementation("org.springframework.boot:spring-boot-starter-test")
````

## 单元测试

在直接测试中，我们直接在 grpc-service bean/实例上调用方法。

> 如果您自己创建的 grpc-service 实例, 请确保您先处理所需的依赖关系。 如果您使用Spring，它会处理您的依赖关系，但作为代价，您必须配置Spring。

### 独立测试

独立测试对外部库没有任何依赖关系(事实上你甚至不需要这个项目)。 然而，没有外部依赖关系并不总能使您的生活更加容易， 您可能需要复制其他库来执行您的行为。 使用 [Mockito](https://site.mockito.org) 这样的模拟库会简化你的流程，因为它限制依赖树的深度。

````java
public class MyServiceTest {

    private MyServiceImpl myService;

    @BeforeEach
    public void setup() {
        myService = new MyServiceImpl();
        OtherDependency foobar = ...; // mock(OtherDependency.class)
        myService.setFoobar(foobar);
    }

    @Test
    void testSayHellpo() throws Exception {
        HelloRequest request = HelloRequest.newBuilder()
                .setName("Test")
                .build();
        StreamRecorder<HelloReply> responseObserver = StreamRecorder.create();
        myService.sayHello(request, responseObserver);
        if (!responseObserver.awaitCompletion(5, TimeUnit.SECONDS)) {
            fail("The call did not terminate in time");
        }
        assertNull(responseObserver.getError());
        List<HelloReply> results = responseObserver.getValues();
        assertEquals(1, results.size());
        HelloReply response = results.get(0);
        assertEquals(HelloReply.newBuilder()
                .setMessage("Hello ==> Test")
                .setCounter(1337)
                .build(), response);
    }

}
````

### 基于Spring的测试

如果您使用Spring来管理您自己的依赖关系，您实际上正在进入集成测试领域。 请确保您不要启动整个应用程序，而只提供所需的依赖关系为 (模拟) 的 Bean 类。

> **注意:** 在测试期间，Spring 不会自动配置所有必须的 Bean。 您必须在有`@Configuration` 注解的类中手动创建它们。

````java
@SpringBootTest
@SpringJUnitConfig(classes = { MyServiceUnitTestConfiguration.class })
// Spring doesn't start without a config (might be empty)
// Don't use @EnableAutoConfiguration in this scenario
public class MyServiceTest {

    @Autowired
    private MyServiceImpl myService;

    @Test
    void testSayHellpo() throws Exception {
        HelloRequest request = HelloRequest.newBuilder()
                .setName("Test")
                .build();
        StreamRecorder<HelloReply> responseObserver = StreamRecorder.create();
        myService.sayHello(request, responseObserver);
        if (!responseObserver.awaitCompletion(5, TimeUnit.SECONDS)) {
            fail("The call did not terminate in time");
        }
        assertNull(responseObserver.getError());
        List<HelloReply> results = responseObserver.getValues();
        assertEquals(1, results.size());
        HelloReply response = results.get(0);
        assertEquals(HelloReply.newBuilder()
                .setMessage("Hello ==> Test")
                .setCounter(1337)
                .build(), response);
    }

}
````

和所需的配置类：

````java
@Configuration
public class MyServiceUnitTestConfiguration {

    @Bean
    OtherDependency foobar() {
        // return mock(OtherDependency.class);
    }

    @Bean
    MyServiceImpl myService() {
        return new MyServiceImpl();
    }

}
````

## 集成测试

然而，您有时需要测试整个调用栈。 例如，如果认证发挥了作用。 但在这种情况下，建议限制您的测试范围，以避免像 空数据库这样可能的外部影响。

在这一点上，不使用 Spring 测试您的 Spring 应用程序是毫无意义的。

> **注意:** 在测试期间，Spring 不会自动配置所有必须的 Bean。 您必须在有 `@Configuration` 注解修饰的类中手动创建他们，或显式的包含相关的自动配置类。

````java
@SpringBootTest(properties = {
        "grpc.server.inProcessName=test", // Enable inProcess server
        "grpc.server.port=-1", // Disable external server
        "grpc.client.inProcess.address=in-process:test" // Configure the client to connect to the inProcess server
        })
@SpringJUnitConfig(classes = { MyServiceIntegrationTestConfiguration.class })
// Spring doesn't start without a config (might be empty)
@DirtiesContext // Ensures that the grpc-server is properly shutdown after each test
        // Avoids "port already in use" during tests
public class MyServiceTest {

    @GrpcClient("inProcess")
    private MyServiceBlockingStub myService;

    @Test
    @DirtiesContext
    public void testSayHello() {
        HelloRequest request = HelloRequest.newBuilder()
                .setName("test")
                .build();
        HelloReply response = myService.sayHello(request);
        assertNotNull(response);
        assertEquals("Hello ==> Test", response.getMessage())
    }

}
````

所需的配置看起来像这样：

````java
@Configuration
@ImportAutoConfiguration({
        GrpcServerAutoConfiguration.class, // Create required server beans
        GrpcServerFactoryAutoConfiguration.class, // Select server implementation
        GrpcClientAutoConfiguration.class}) // Support @GrpcClient annotation
public class MyServiceIntegrationTestConfiguration {

    @Bean
    OtherDependency foobar() {
        return ...; // mock(OtherDependency.class);
    }

    @Bean
    MyServiceImpl myServiceImpl() {
        return new MyServiceImpl();
    }

}
````

> 注意：这个代码看起来可能比单元测试更短/更简单，但执行时间要长一些。

## gRPCurl

[`gRPCurl`](https://github.com/fullstorydev/grpcurl) 是一个小型的命令行应用程序， 您可以在应用程序运行时查询。 或者如它们的 Readme 说的那样：

> 对于 gRPC 服务，它基本上是 `curl` 工具

您甚至可以使用 `jq` 工具来处理的响应，将它用于自动化测试中。

如果您已经知道要查询什么，跳过第一/这个部分。

````bash
$ # First scan the server for available services
$ grpcurl --plaintext localhost:9090 list
net.devh.boot.grpc.example.MyService
$ # Then list the methods available for that call
$ grpcurl --plaintext localhost:9090 list net.devh.boot.grpc.example.MyService
net.devh.boot.grpc.example.MyService.SayHello
$ # Lets check the request and response types
$ grpcurl --plaintext localhost:9090 describe net.devh.boot.grpc.example.MyService/SayHello
net.devh.boot.grpc.example.MyService.SayHello is a method:
rpc SayHello ( .HelloRequest ) returns ( .HelloReply );
$ # Now we only have query for the request body structure
$ grpcurl --plaintext localhost:9090 describe net.devh.boot.grpc.example.HelloRequest
net.devh.boot.grpc.example.HelloRequest is a message:
message HelloRequest {
  string name = 1;
}
````

> 注意： `gRPCurl` 支持 `.` 和 `/` 作为服务名称和方法名称之间的分隔符：
>
> - `net.devh.boot.grpc.example.MyService.SayHello`
> - `net.devh.boot.grpc.example.MyService/SayHello`
>
> 我们推荐第二种方式，因为它跟 grpc 的内部全方法名称匹配，并且方法名称在调用中更容易识别到。

````bash
$ # Finally we can call the actual method
$ grpcurl --plaintext localhost:9090 net.devh.boot.grpc.example.MyService/SayHello
{
  "message": "Hello ==> ",
  "counter": 1337
}
$ # Or call it with a populated request body
$ grpcurl --plaintext -d '{"name": "Test"}' localhost:9090 net.devh.boot.grpc.example.MyService/SayHello
{
  "message": "Hello ==> Test",
  "counter": 1337
}
````

> 注意：如果您使用了 window 终端或想要在数据块中使用变量，那么您必须使用 `"` 而不是 `'`，并转义实际json中的 `"` 。
>
> ````cmd
>
> > grpcurl --plaintext -d "{\"name\": \"Test\"}" localhost:9090 net.devh.boot.grpc.example.MyService/sayHello
    {
      "message": "Hello ==> Test",
      "counter": 1337
    }
    ````

更多 `gRPCurl` 的信息，请参阅他们的 [官方文档](https://github.com/fullstorydev/grpcurl)

## 附加主题 <!-- omit in toc -->

- [入门指南](getting-started.md)
- [配置](configuration.md)
- [异常处理](exception-handling.md)
- [上下文数据 / Bean 的作用域](contextual-data.md)
- *测试服务*
- [服务端事件](events.md)
- [安全性](security.md)

----------

[<- 返回索引](../index.md)
