# dubbo 简单使用
dubbo 作为一个 [RPC 框架](distributed/RPC.md) ，肯定是需要服务提供端、服务消费端、公用 api。

## 定义公用api
新建 maven 工程，并添加公共接口，并打成 jar包
```java
public interface IHelloService {
    String sayHello(String name);
}
```

## 新建 provider 模块
**新建 SpringBoot 项目，增加dubbo、zk、公共api 依赖**

```xml
<dependency>
    <groupId>org.apache.dubbo</groupId>
    <artifactId>dubbo-spring-boot-starter</artifactId>
    <version>2.7.8</version>
</dependency>
<dependency>
    <groupId>org.apache.dubbo</groupId>
    <artifactId>dubbo-dependencies-zookeeper</artifactId>
    <version>2.7.8</version>
</dependency>
<dependency>
    <groupId>com.study</groupId>
    <artifactId>common-api</artifactId>
    <version>1.0</version>
</dependency>
```

**新增公用接口实现类**
```java
@DubboService
public class HelloService implements IHelloService {
    @Override
    public String sayHello(String name) {
        return "hello: " + name;
    }
}
```

**Spring Boot 启动类增加 dubbo 注解**
```java
@SpringBootApplication
@DubboComponentScan(basePackages = "com.study.provider.service")
public class DubboProviderDemoApplication {

    public static void main(String[] args) {
        SpringApplication.run(DubboProviderDemoApplication.class, args);
    }

}
```

**application.properties 增加dubbo相关配置**
```properties
dubbo.application.name=dubbo-provider-demo
dubbo.registry.address=zookeeper://localhost:2181
```

**以上步骤完成后启动项目**

## 新建 consumer 模块

**新建 SpringBoot 项目，增加dubbo、zk、公共api 依赖**

```xml
<dependency>
    <groupId>org.apache.dubbo</groupId>
    <artifactId>dubbo-spring-boot-starter</artifactId>
    <version>2.7.8</version>
</dependency>
<dependency>
    <groupId>org.apache.dubbo</groupId>
    <artifactId>dubbo-dependencies-zookeeper</artifactId>
    <version>2.7.8</version>
</dependency>
<dependency>
    <groupId>com.study</groupId>
    <artifactId>common-api</artifactId>
    <version>1.0</version>
</dependency>
```

**增加服务消费代码**
```java
@RestController
@SpringBootApplication
public class DubboConsumerDemoApplication {

    public static void main(String[] args) {
        SpringApplication.run(DubboConsumerDemoApplication.class, args);
    }

    @DubboReference
    private IHelloService helloService;

    @GetMapping("hello")
    public String hello(String name) {
        System.out.println(name);
        return helloService.sayHello(name);
    }
}
```
**application.properties 增加dubbo相关配置**
```properties
dubbo.application.name=dubbo-consumer-demo
dubbo.registry.address=zookeeper://localhost:2181
server.port=8888
```
**启动项目，并打开浏览器访问地址**

至此，简单demo使用以及完成了。