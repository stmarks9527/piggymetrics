[![Build Status](https://travis-ci.org/sqshq/PiggyMetrics.svg?branch=master)](https://travis-ci.org/sqshq/PiggyMetrics)
[![codecov.io](https://codecov.io/github/sqshq/PiggyMetrics/coverage.svg?branch=master)](https://codecov.io/github/sqshq/PiggyMetrics?branch=master)
[![GitHub license](https://img.shields.io/github/license/mashape/apistatus.svg)](https://github.com/sqshq/PiggyMetrics/blob/master/LICENCE)
[![Join the chat at https://gitter.im/sqshq/PiggyMetrics](https://badges.gitter.im/sqshq/PiggyMetrics.svg)](https://gitter.im/sqshq/PiggyMetrics?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=badge)

# 量化指标 

Piggy Metrics是一个简单的财务顾问应用程序，使用SpringBoot、SpringCloud和Docker来演示微服务架构模式。[该项目旨在作为一个教程，但欢迎您分叉它和把它变成其他东西！](http://martinfowler.com/microservices/)

![](https://cloud.githubusercontent.com/assets/6069066/13864234/442d6faa-ecb9-11e5-9929-34a9539acde0.png)
![Piggy Metrics](https://cloud.githubusercontent.com/assets/6069066/13830155/572e7552-ebe4-11e5-918f-637a49dff9a2.gif)

## 功能服务 

Piggy度量被分解为三个核心微服务。它们都是围绕某些业务域组织的可独立部署的应用程序。

#### 账户服务 

包含一般输入逻辑和验证：收入/支出项、储蓄和帐户设置。

Method	| Path	| Description	| User authenticated	| Available from UI
------------- | ------------------------- | ------------- |:-------------:|:----------------:|
GET	| /accounts/{account}	| Get specified account data	|  |
GET	| /accounts/current	| Get current account data	| × | ×
GET	| /accounts/demo	| Get demo account data (pre-filled incomes/expenses items, etc)	|   | 	×
PUT	| /accounts/current	| Save current account data	| × | ×
POST	| /accounts/	| Register new account	|   | ×
  -------------------------------------------------------------------------------------------------------

#### 统计服务 

对主要统计参数执行计算，并捕获每个帐户的时间序列。数据点包含标准化为基本货币和时间段的值。此数据用于跟踪帐户生命周期内的现金流动态。

Method	| Path	| Description	| User authenticated	| Available from UI
------------- | ------------------------- | ------------- |:-------------:|:----------------:|
GET	| /statistics/{account}	| Get specified account statistics	          |  |
GET	| /statistics/current	| Get current account statistics	| × | ×
GET	| /statistics/demo	| Get demo account statistics	|   | ×
PUT	| /statistics/{account}	| Create or update time series datapoint for specified account	|   |
-------------------------------------------------------------------------------------------------

#### 通知服务 

存储用户联系信息和通知设置（提醒、备份频率等）。计划的工作人员从其他服务收集所需的信息，并向订阅的用户发送电子邮件。

Method	| Path	| Description	| User authenticated	| Available from UI
------------- | ------------------------- | ------------- |:-------------:|:----------------:|
GET	| /notifications/settings/current	| Get current account notification settings	| × | ×
PUT	| /notifications/settings/current	| Save current account notification settings	| × | ×
---------------------------------------------------------------------------------

#### 备注 

-   每个微服务都有自己的数据库，因此无法绕过API直接访问持久性数据。
-   MongoDB用作每个服务的主数据库。
-   所有服务都通过Rest API相互通信

## 基础架构 

SpringCloud为开发人员提供了快速实现常见分布式系统模式的强大工具-###Config服务SpringCloud
Config是面向分布式系统的水平可扩展集中式配置服务。[它使用可插入的存储库层，当前支持本地存储、Git和Subversion。](http://cloud.spring.io/spring-cloud-config/spring-cloud-config.html)

在这个项目中，我们将使用本机配置文件，它只是从本地类路径加载配置文件。``您可以在Config服务资源中看到共享目录。``现在，当Notification-service请求其配置时，Config服务响应shared/notification-service.yml和shared/application.yml（在所有客户端应用程序之间共享）。
````

##### 客户端使用

只需构建具有spring-cloud-starter-config依赖关系的SpringBoot应用程序，其余部分由自动配置完成。
``

现在，您的应用程序中不需要任何嵌入的属性。`只需为bootstrap.yml提供应用程序名称和配置服务URL：
`



```yml
spring:
  application:
    name: notification-service
  cloud:
    config:
      uri: http://config:8888
      fail-fast: true
```


##### 使用SpringCloud配置，您可以动态更改应用程序配置。 

[例如，使用\@Refresh
Scope注释电子邮件服务bean。](https://github.com/sqshq/PiggyMetrics/blob/master/notification-service/src/main/java/com/piggymetrics/notification/service/EmailServiceImpl.java)这意味着您可以更改电子邮件文本和主题，而无需重建和重新启动通知服务。
``

首先，在Config服务器中更改所需的属性。然后对通知服务进行刷新调用：curl-H"授权：承载#token#"-XPOST
http://127.0.0.1：8000/notifications/refresh ``

您还可以使用存储库Web挂接自动执行此过程

##### 备注 

-   ``\@刷新范围不能与\@配置类一起使用，也不会忽略\@计划的方法 ````
-   `fail-fast属性表示如果Spring Boot应用程序无法连接到配置服务，它将立即启动失败。
    `

### 认证服务 

授权责任被提取到一个单独的服务器，该服务器为后端资源服务授予OAuth2令牌。[AuthServer用于用户授权，也可用于边界内的安全机器对机器通信。](https://tools.ietf.org/html/rfc6749)

[``](https://tools.ietf.org/html/rfc6749#section-4.3)在此项目中，我使用密码凭据授予类型进行用户授权（因为它仅由UI使用），并使用客户端凭据授予进行服务到服务通信。
[``](https://tools.ietf.org/html/rfc6749#section-4.4)

SpringCloudSecurity提供了方便的注释和自动配置，使其在服务器端和客户端都可以轻松实现。[您可以在文档中了解有关这方面的更多信息。](http://cloud.spring.io/spring-cloud-security/spring-cloud-security.html)

在客户端，一切都与传统的基于会话的授权完全一样。``您可以从请求中检索Principal对象，使用基于表达式的访问控制和\@PreAuthorize注释检查用户角色。
``

每个Piggy
Metrics客户端都有一个作用域：后端服务的服务器和浏览器的ui。````我们可以使用\@Pre
Authorize注释保护控制器免受外部访问： ``

    @预授权（“#oauth2.has Scope（‘server’）”）@请求映射（值=“accounts/{name}”，方法=请求方法。
    GET）公共列表<DataPoint>按帐户名获取统计信息（@路径变量字符串名称）{返回统计信息Service.find按帐户名（名称）；}


### API网关 

API网关是系统中的单个入口点，用于处理请求并将其路由到适当的后端服务，或者通过聚合散集调用的结果。[此外，它还可用于身份验证、洞察、压力和金丝雀测试、服务迁移、静态响应处理和主动流量管理。](http://techblog.netflix.com/2013/01/optimizing-netflix-api.html)

Netflix提供这样的边缘服务，SpringCloud允许将其与单个\@Enable Zuul
Proxy注释一起使用。[在这个项目中，我们使用Zuul来存储一些静态内容（UI应用程序），并路由请求以适应微服务。](http://techblog.netflix.com/2013/06/announcing-zuul-edge-service-in-cloud.html)下面是通知服务的简单的基于前缀的路由配置：
``



    zuul：路由：通知服务：路径：/通知/**服务ID：通知服务条前缀：false


这意味着所有以/notifications开头的请求都将路由到Notification服务。``如您所见，没有硬编码地址。Zuul使用服务发现机制查找通知服务实例以及断路器和负载平衡器，如下所述。

### 服务发现 

"服务发现"允许自动检测所有注册服务的网络位置。由于自动扩展、故障或升级，这些位置可能具有动态分配的地址。

服务发现的关键部分是注册表。在这个项目中，我们使用Netflix
Eureka。Eureka是客户端发现模式的一个很好的例子，其中客户端负责查找可用服务实例的位置并在它们之间进行负载平衡。

使用Spring
Boot，您可以使用Spring-cloud-starter-eureka-server依赖项\@Enable Eureka
Server注释和简单配置属性轻松构建Eureka注册表。 ````

使用\@Enable Discovery
Client批注启用了客户端支持，bootstrap.yml使用应用程序名称： ````


    弹簧：应用程序：名称：通知服务

该服务将在Eureka服务器上注册，并提供元数据，如主机、端口、运行状况指示器URL、主页等。Eureka从属于服务的每个实例接收心跳消息。如果心跳超过可配置的时间表，实例将从注册表中删除。

此外，Eureka提供了一个简单的界面，您可以在其中跟踪正在运行的服务和许多可用实例：http://localhost：8761
``

### 负载平衡器、断路器、Http客户端 

#### 功能区 

Ribbon是一个客户端负载平衡器，它使您可以很好地控制HTTP和TCP客户端的行为。与传统的负载平衡器相比，不需要额外的网络跳-您可以直接联系所需的服务。

开箱即用，可与SpringCloud和服务发现进行本机集成。[Eureka
Client提供了可用服务器的动态列表，因此Ribbon可以在它们之间进行平衡。](https://github.com/sqshq/PiggyMetrics#service-discovery)

#### 海斯特里克斯 

Hystrix是断路器模式的实现，它使我们在与其他服务通信时可以控制延迟和网络故障。[主要想法是停止分布式环境中的级联故障-这有助于快速故障并尽快恢复-容错系统的重要方面可以自我修复。](http://martinfowler.com/bliki/CircuitBreaker.html)

此外，Hystrix会为每个命令生成有关执行结果和延迟的指标，我们可以使用这些指标来监视系统的行为。

#### 费恩 

Feign是一个声明式Http客户端，它与Ribbon和Hystrix无缝集成。`实际上，单一的Spring-cloud-starter-feign依赖和@Enable Feign Clients注释为我们提供了一套完整的工具，包括负载平衡器、断路器和Http Client，这些工具具有合理的默认配置。
```

下面是帐户服务的一个示例：

    @Feign Client（名称=“statistics-service”）公共接口统计服务Client{@请求映射（方法=请求方法。
    PUT，值=“/statistics/{account Name}”，消耗=介质类型。

    APPLICATION_JSON_UTF8_VALUE）无效更新统计信息（@路径变量（“帐户名”）字符串帐户名，帐户帐户）；}


-   您所需要的一切都只是一个界面
-   `您可以在Spring MVC控制器和Feign方法之间共享@请求映射部分
    `
-   由于通过Eureka自动发现，以上示例仅指定了所需的服务ID-统计信息-服务
    ``

### 监视器控制面板 

在此项目配置中，每个带有Hystrix的微服务都通过SpringCloud总线（带有AMQP代理）将度量推送到涡轮机。[Monitoring项目只是一个带有涡轮机和Hystrix控制面板的小型Spring引导应用程序。](https://github.com/Netflix/Turbine)

让我们来看一下我们的系统在负载下的行为：StatisticsService在请求处理过程中会模拟一个延迟。响应超时设置为1秒：

  --------------------------------------------------------------------------------------- ----------------------------------------------------------------------------------------------- --------------------------------------------------------------------------------- --------------------------------------------------------------------------------------
  `0毫秒延迟                                                                              `500ms延迟                                                                                      `800ms延迟                                                                        `1100ms延迟
  `                                                                                       `                                                                                               `                                                                                 `

  系统性能良好。吞吐量约为22rps。统计信息服务中的活动线程数很少。中位服务时间约为50ms。   活动线程的数量正在增加。我们可以看到紫色数量的线程池拒绝，因此大约40%的错误，但电路仍然关闭。   半开状态：命令失败率大于50%，断路器启动。在睡眠窗口时间量之后，下一个请求通过。   100%的请求失败。电路现在永久断开。休眠时间后重试不会再次关闭电路，因为单个请求太慢。
  --------------------------------------------------------------------------------------- ----------------------------------------------------------------------------------------------- --------------------------------------------------------------------------------- --------------------------------------------------------------------------------------

### 日志分析 

在尝试识别分布式环境中的问题时，集中式日志记录非常有用。使用Elasticsearch、Logstash和Kibana堆栈，您可以轻松搜索和分析日志、利用率和网络活动数据。

### 分布式跟踪 

分析分布式系统中的问题可能很困难，特别是尝试跟踪从一个微服务传播到另一个微服务的请求。

[SpringCloudSleuth通过提供对分布式跟踪的支持来解决这个问题。](https://cloud.spring.io/spring-cloud-sleuth/)它将两种类型的ID添加到日志记录中：traceId和spanId。`spanId表示基本的工作单元，例如发送HTTP请求。`跟踪ID包含一组跨距，构成树形结构。`例如，对于分布式大数据存储，跟踪可能由PUT请求构成。``为每个操作使用traceId和spanId，我们知道应用程序在处理请求时的时间和位置，从而使读取日志更加容易。
`````

日志如下，请注意Slf4J MDC中的\[appname、trace Id、span
Id、可导出\]条目： ``

    2018-07-2623：13：49.381WARN[网关，3216d0de1384bb4f，3216d0de1384bb4f，false]2999---[nio-4000-exec-1]o.s.c.n.z.f.r.s.
    抽象功能区命令：命令帐户服务的Hystrix超时20000ms设置为低于功能区读取和连接超时的组合80000ms。2018-07-2623：13：49.562INFO[account-service，3216d0de1384bb4f，404ff09c5cf91d2e，false]3079---[nio-6000-exec-1]c.p.account.service。帐户服务实施：已创建新帐户：测试

-   ``appname：从属性spring.application.name记录范围的应用程序的名称 ``
-   `跟踪ID：这是分配给单个请求、作业或操作的ID
    `
-   `span ID：发生的特定操作的ID
    `
-   `可导出：是否将日志导出到Zipkin
    `

## 基础架构自动化 

部署微服务，与其相互依赖，是比部署单一应用程序复杂得多的过程。拥有一个完全自动化的基础架构是非常重要的。使用连续交付方法，我们可以实现以下好处：

-   随时发布软件的能力
-   任何生成都可能最终成为版本
-   一次生成工件-根据需要进行部署

下面是一个简单的连续交付工作流程，在此项目中实现：

[在此配置中，Travis
CI会为每个成功的git推送生成标记图像。](https://github.com/sqshq/PiggyMetrics/blob/master/.travis.yml)因此，DockerHub上的每个微服务总是有最新的图像，而旧的图像则用gitcommit哈希标记。``如果需要，可以轻松部署其中的任何一个并快速回滚。

## 让我们来试试 

请注意，启动8个Spring Boot应用程序、4个Mongo DB实例和一个Rabbit
Mq至少需要4Gb RAM。

#### 启动前 

-   安装Docker和Docker组件。
-   更改.env文件中的环境变量值以获得更高的安全性，或者保持原样。 ``
-   生成项目：mvn包\[-跳过测试\] ``

#### 生产模式 

在此模式下，所有最新的图像都将从Docker
Hub中提取。`只需复制docker-compose.yml并点击docker-compose即可
```

#### 开发模式 

如果您希望自己构建映像，则必须克隆存储库并使用maven构建工件。之后，向上运行docker-compose-f
docker-compose.yml-f docker-compose.dev.yml ``

`docker-compose.dev.yml继承了docker-compose.yml，还可以在本地生成映像，并公开所有容器端口，以便于开发。
```

如果要在Intellij
Idea中启动应用程序，则需要使用Env文件插件或手动导出.env文件中列出的环境变量（确保已导出：printenv）
````

#### 重要端点 

-   http://localhost：80-网关
-   http://localhost：8761-Eureka控制面板
-   http://localhost：9000/hystrix-Hystrix控制面板（涡轮流链接：http://turbine-stream-service：8080/turbine/turbine.stream）
    ``
-   http://localhost：15672-Rabbit Mq管理（默认登录名/密码：来宾/来宾）

## 欢迎投稿！ 

Piggy Metrics是开源的，非常感谢您的帮助。请随时提出建议并实施任何改进。
