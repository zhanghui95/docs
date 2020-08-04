

# Spring Cloud Alibaba-Naocs篇

## 1. 用Nacos作为服务注册中心与配置中心

1. 下载二进制包、解压：https://github.com/alibaba/nacos/releases

2. 启动服务器

   * 在**Linux / Unix / Mac**平台上，运行以下命令以独立模式启动服务器：

   ```
   sh startup.sh -m standalone
   ```

   * 在**Windows**平台上，运行以下命令以独立模式启动服务器。或者，您也可以双击startup.cmd以运行NacosServer

   ```
   cmd startup.cmd -m standalone
   ```

3. 进入控制台界面

``` 
浏览器输入: http://localhost:8848/nacos即可
高版本控制台默认账号密码：nacos/nacos
```

## 2. 启用mysql持久化配置

启动nacos并成功进入到控制台证明nacos注册中心与配置中心已经ok了，有一个问题，就是默认启动的nacos将来在作为配置中心时会有很多的配置文件，但是它是基于文件夹下file存储的，不能很好的移植或集群，所以需要先通过数据持久化到数据库，才能保证后面更好的玩耍。

大致流程为创建数据库，导入nacos提供的mysql建表sql脚本，打开下载好的nacos-server -> conf -> application.properties文件 配置数据库连接信息等

``` properties
spring.datasource.platform=mysql
db.num=1
db.url.0=jdbc:mysql://localhost:3306/nacos_config?characterEncoding=utf8&connectTimeout=1000&socketTimeout=3000&autoReconnect=true
db.user=root
db.password=root
```

> 具体可参考 http://nacos.io 官方介绍

通过上面的配置，nacos持久化到数据库就基本完成了，接下来在控制台创建的配置文件与历史记录将会存储到mysql中。

## 3. 其他项目注册到nacos

1. pom引入nacos服务发现等依赖

```xml
 <!-- spring-cloud -->
<dependency>
		<groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-config</artifactId>
</dependency>
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
</dependency>
```

2. application.properties添加配置

```properties
server.port=9001
spring.profiles.active=dev
spring.application.name=auth

spring.cloud.nacos.config.prefix=${spring.application.name}
spring.cloud.nacos.config.enabled=true
spring.cloud.nacos.config.server-addr=127.0.0.1:8848
spring.cloud.nacos.discovery.server-addr=127.0.0.1:8848
spring.cloud.nacos.config.file-extension=properties
```

## 4. 使用nacos作为配置中心

> 通过上一步配置，启动项目后服务已经注册到nacos了，实现了服务注册发现。接下来使用配置中心。

1. 上一步配置的spring.cloud.nacos.config.file-extension=properties其实是指定nacos中配置文件的格式，我这里用properties，也可指定yml格式，那么在nacos中新建的文件格式就要一直。
2. 打开http://localhost:8848/nacos 账号密码：nacos/nacos
3. 在配置列表菜单，点击新建。**Data ID** 这里指定应用名例如 auth-dev.properties，默认**Group**不需要更改，格式选择properties，配置内容就是填写一般在application.properties中的配置。(说明：1. Data ID这里auth为应用名，-dev为dev环境，可以不指定则不用激活dev，.properties为格式，如果指定yml，则这里是 .yml 这里是一个坑 必须有文件格式 要不然服务启动默认找配置中心配置是找不到的)
4. 发布后，应用主类添加 `@EnableDiscoveryClient` 允许服务发现注解，启动服务，则默认会从配置中心获取配置，完成服务注册发现与配置中心。

## 5. 我的进度

- [x] nacos服务注册配置中心mysql持久化
- [x] gateway实现动态路由刷新
- [x] 服务间Dubbo调用并注册nacos
- [x] 动态数据源+seata分布式事务AT模式+MybatisPlus(Seata交流群管理对动态数据源的PR)
- [x] sentinel熔断 限流
- [x] sentinel配置实现nacos控制
- [x] 集成swagger增强工具knife4j
- [x] 集成JustAuth适配第三方登录
- [ ] ~~集成shaun(基于pac4j的安全框架)~~
- [x] 集成JwtPermission实现token认证与单点登录