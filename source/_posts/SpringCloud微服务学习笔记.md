---
title: SpringCloud微服务学习笔记
date: 2024-12-06 15:47:36
tags: studyNote
---

## SpringCloud

### 拆分微服务

- 每个微服务在配置文件中都需要设置新的名字。
- Swagger配置的时候也要修改扫描controller的路径。
- 主类的mapper包扫描也要修改

### 远程调用

- 使用`RestTemplate`远程微服务调用其他资源，利用网络。

### 服务治理

#### 注册中心原理

负载均衡： 轮询、随机、加权轮询、哈希。

注册中心提供服务：

- 记录并监控服务提供者状态。
- 心跳续约（服务提供者）
- 推送变更（服务消费者）

#### nacos注册中心

```PowerShell
docker run -d \
--name nacos \
--env-file ./nacos/custom.env \
-p 8848:8848 \
-p 9848:9848 \
-p 9849:9849 \
--network=hmall \
--restart=always \
nacos/nacos-server:v2.1.0-slim
```

##### 服务注册 -> 服务发现

#### OpenFeign

- 底层完成跨服务的http请求代码，`feginClient` 注册一个`client`接口，直接调用

#### 实现原理

### 网关路由配置

- 网关作为另一新的微服务，依赖于`spring-cloud-gateway`.
- 在配置文件里对`routes` `nacos` `server-port`进行配置。

#### 路由（routes）的配置

- 源码内部其实是`RoutesDefinition`
- `predicates` `filter` 等等基本属性

#### 网关登录校验

在网关处对JWT令牌进行校验。在网关转发之前，对JWT进行校验。

- 网关内部过滤器实际上分为`pre` `post` 两部分，当进入微服务得到请求结果之后，出请求时也会经过`post`部分。
- 网关进行登录校验后向微服务传递登录成功的用户信息，用户信息的保存 -> HTTP HEADER

##### 自定义过滤器

- `GateWayFilter` 作用于指定的路由，在实际实现中，使用继承抽象类`AbstractGatewayFilterFactory`，进行自定义。使用**装饰模式类** `OrderedGatewayFilter`对优先级进行配置。

  - 返回一个新的 `GateWayFilter`（interface）只能通过创建匿名内部类。

  ```java
  public class PrintAnyGatewayFilterFactory extends AbstractGatewayFilterFactory<PrintAnyGatewayFilterFactory.Config> {
      @Override
      public GatewayFilter apply(Config config) {
          return new OrderedGatewayFilter(new GatewayFilter() {
              @Override
              public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
                  System.out.println("自定义打印过滤器已经执行");
                  // 放行
                  return chain.filter(exchange);
              }
          }, 1);
      }
      @Data
      public static class Config {
          private String a;
          private String b;
          private String c;
      }
      @Override
      public List<String> shortcutFieldOrder() {
          // 确定参数顺序
          return List.of("a", "b", "c");
      }
      public PrintAnyGatewayFilterFactory() {
          // 将参数类传递给父类
          super(Config.class);
      }
  }
  ```

  

  - 当拥有多个参数时，自定义一个配置属性类，同时将参数对应的顺序依次返回。

  

- `GlobalFilter` 作用范围于所有的路由

##### 网关向用户传递用户信息

使用`springMVC`将定义一个通用的拦截器，所有微服务引入通用模块，拦截器将用户信息保存在`ThreadLocal`.

在公共模块中，添加`interceptors`获得网关传递过来的用户信息id，`spring-mvc`的配置，还要添加好`interceptors`,但由于网关内部底层不是由`spring-mvc`实现，所需要添加配置条件注解，借助`springmvc`核心类`DispatcherServlet`。排除掉无法在网关模块中自动装配的错误。 

```java
@Configuration
@ConditionalOnClass(DishpatcherServlet.class)
```

##### 用户间传递信息OpenFeign

使用OpenFeign提供的拦截器接口，建立拦截器，加入并保存请求头中的用户信息。`RequsetInterceptors`

#### 总结：微服务项目登录以及用户信息传递解决方案

- 网关获取用户信息
  - 使用网关提供的`GlobalFilter`实施登录检验，和保存请求头。
- 网关向微服务内部传递用户信息
  -  使用`HandlerInterceptor`也就是SpringMVC配置拦截器，将网关发来的请求头中携带的用户信息，保存到`ThreadLocal`
  -  在所有微服务中的公共类进行拦截器配置。（注意要排除掉网关模块的影响）
- 微服务之间传递用户信息
  - 使用OpenFeign提供的`RequestInterceptor`拦截器，将本地`ThreadLocal`中的用户信息，添加到请求头中发出
  - 接收请求的微服务，通过MVC拦截器，保存用户信息。

### 配置管理

微服务读取**配置管理服务**中的配置，**共享**配置

#### 配置共享

nacos实现配置共享

#### 配置热更新

- nacos需创建一个和微服务名有关的配置文件`cart-service-local.yml`
- 创建一个`ConfigurationProperties`利用类，在代码中更新写死的需要热改变的配置文件

#### 动态路由

动态路由配置 

- 添加监听器

```java
@Component
@RequiredArgsConstructor
@Slf4j
public class DynamicRouteLoader {

    private final NacosConfigManager nacosConfigManager;

    private final String dataId =  "gateway-routes.json";
    private final String group =  "DEFAULT_GROUP";
    private final RouteDefinitionWriter routeDefinitionWriter;

    private final Set<String> routeIds = new HashSet<>();

    @PostConstruct
    public void initRouteConfigListener() throws NacosException {
        // 项目初始化的时候先拉取配置文件的同时添加一个监听器
        String configInfo = nacosConfigManager.getConfigService()
                .getConfigAndSignListener(dataId, group, 5000, new Listener() {
                    @Override
                    public Executor getExecutor() {
                        return null;
                    }

                    @Override
                    public void receiveConfigInfo(String configInfo) {
                        // 监听到配置的变更，此时更新路由表
                        log.info("监听到配置的变更，此时更新路由表....");
                        updateConfigInfo(configInfo);
                    }
                });
        // 第一次拉取到配置的时候也要更新路由表
        updateConfigInfo(configInfo);
    }

    public void updateConfigInfo(String configInfo) {
        // 解析配置信息，转换为路由RouteDefinition
        // 传过来的应该是列表
        List<RouteDefinition> routeDefinitions = JSONUtil.toList(configInfo, RouteDefinition.class);
        // 更新路由列表
        // 首先先删除
        for (String routeId : routeIds) {
            routeDefinitionWriter.delete(Mono.just(routeId)).subscribe();
        }
        // 清除记录
        routeIds.clear();
        for (RouteDefinition routeDefinition : routeDefinitions) {
            // 更新路由表
            routeDefinitionWriter.save(Mono.just(routeDefinition)).subscribe();
            // 记录路由id，便于下次更新时进行删除
            routeIds.add(routeDefinition.getId());
        }
    }
}
```

- nacos添加路由配置文件（因为无法对yml文件进行解析，使用json进行配置）

```json
[
    {
        "id": "item",
        "predicates": [{
            "name": "Path",
            "args": {"_genkey_0":"/items/**", "_genkey_1":"/search/**"}
        }],
        "filters": [],
        "uri": "lb://item-service"
    },
    {
        "id": "cart",
        "predicates": [{
            "name": "Path",
            "args": {"_genkey_0":"/carts/**"}
        }],
        "filters": [],
        "uri": "lb://cart-service"
    },
    {
        "id": "user",
        "predicates": [{
            "name": "Path",
            "args": {"_genkey_0":"/users/**", "_genkey_1":"/addresses/**"}
        }],
        "filters": [],
        "uri": "lb://user-service"
    },
    {
        "id": "trade",
        "predicates": [{
            "name": "Path",
            "args": {"_genkey_0":"/orders/**"}
        }],
        "filters": [],
        "uri": "lb://trade-service"
    },
    {
        "id": "pay",
        "predicates": [{
            "name": "Path",
            "args": {"_genkey_0":"/pay-orders/**"}
        }],
        "filters": [],
        "uri": "lb://pay-service"
    }
]
```

### 服务保护和分布式事务

#### 雪崩问题

- 微服务相互调用，服务提供者出现了故障或阻塞。
- 服务调用者没有做好异常处理，导致自身故障。
- 调用链中的所有服务级联失败，导致整个集群故障。

解决（**服务保护**）方案

- 请求限流（保护提供者）
- **线程隔离**（保护消费者）
- 服务熔断（监控已发出的请求，若该接口异常请求比例或慢调用比例，超出了设定的阈值，直接熔断该业务接口）
  - 熔断期间，全部走`fallback `逻辑，返回。

#### Sentinel

簇点链路: 一系列的调用的接口，形成的一条链路。

实现限流规则：在簇点链路中，对每一个接口更改流控中的**单机规则**。

实现服务熔断：需要将`FeignClient`中的作为Sentinel的簇点资源（配置Fallback）。

两种方式：

- FallbackFactory（可以处理远程调用）
- FallbackClass（无法处理远程调用）

- 自定义类需要的client的`FallbackFactory`

##### 服务熔断

###### 断路器

由断路器记录统计服务调用的异常比例和慢请求比例，超过一定阈值开始熔断该服务。

内部由三种状态形成的状态机， `Closed` `Open` `Half-Open`

- 当熔断时间到 `Open` -> `Half-Open` 放行一次请求，根据统计请求状态转换到相应的状态。

#### 分布式事务

##### Seata架构

- **TC**(Transaction Cooridinator)事务协调者，维护全局的和分支的事务的状态，协调全局事务提交或者回滚
- TM(Transaction Manager)事务管理器 ：告知TC，指明全局事务的范围，开始全局事务、提交或回滚全局事务。
- RM(Resource Manager) 资源管理器：管理全局事务下的分支事务，与TC协调以注册分支事务和报告分支事务的状态。

Seata本身就是一个微服务，将其配置修改，docker部署之后，就会注册到nacos服务列表中一起集中管理。

##### XA模式

XA，一种分布式事务处理标准。（**两阶段提交**）

###### 一阶段

- TM报告TC开启全局事务。
- TM开始调用各分支。
- RM向TC注册分支事务。
- 分支微服务执行业务SQL。
- 分支（执行结束之后）报告事务状态。

###### 二阶段

- TM报告TC（所有分支事务结束）全局事务结束。
- TC检查各分支事务状态。向各分支发送指令（**提交/回滚**）

###### 优点

- 满足事务的强一致性，满足ACID原则（Atomicity Consistency Isolation Durability）
- 实现简单

###### 缺点

- 因为在一阶段需要**锁定数据库资源**，（各分支是串行执行的）等待到二阶段才会释放资源。
- 依赖于关系型数据库的实现事务.

##### AT模式

###### 一阶段

- RM注册分支事务
- **RM记录undo-log（数据快照）**
- RM执行业务sql并提交事务。
- RM向TC报告事务状态

###### 二阶段

- TC收到TM全局事务完成信号。
- TC检查分支事务状态。（若需要回滚，**恢复快照数据**，成功，删除快照）。

- **创建一个更新前后数据库快照，不用等待各自业务sql的执行完成后释放锁资源。**
- **最终一致性**：只有在更新恢复快照之前，会有短暂的时间出现数据的不一致性。

##### AT vs XA

- AT使用数据快照实现数据回滚，XA模式依赖于数据库机制实现回滚。
- XA一阶段不提交事务，锁定资源，AT一阶段提交事务，不锁定资源。



