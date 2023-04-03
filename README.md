# 微服务容器化改造

## 改造前的微服务

- [gateway](https://github.com/xdevops-caj-lab-cloudnative-tk/PiggyMetrics/tree/master/gateway)

## 每个微服务有自己的代码仓库

将原来微服务属于一个Maven module改为每个微服务有自己的代码仓库。

添加`.gitignore`文件，以忽略一些不需要提交到代码仓库的文件。

在pom.xml中将parent直接改为spring-boot-starter-parent，并引入spring-cloud-dependencies。

```xml
	<parent>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-parent</artifactId>
		<version>2.0.3.RELEASE</version>
		<relativePath/> <!-- lookup parent from repository -->
	</parent>

	<properties>
		<project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
		<spring-cloud.version>Finchley.RELEASE</spring-cloud.version>
		<java.version>1.8</java.version>
	</properties>

	<dependencyManagement>
		<dependencies>
			<dependency>
				<groupId>org.springframework.cloud</groupId>
				<artifactId>spring-cloud-dependencies</artifactId>
				<version>${spring-cloud.version}</version>
				<type>pom</type>
				<scope>import</scope>
			</dependency>
		</dependencies>
	</dependencyManagement>
```

指定groupId:

```xml
<groupId>com.piggymetrics</groupId>
```

## 本地构建运行

```bash
mvn clean install && mvn spring-boot:run
```

此时发现报错，因此程序试图从Spring Cloud Config Server拉取应用配置。

## 去除从Spring Cloud Config Server拉取配置

将`src/main/resources`下的bootstrap.yml改为application.yaml。

并移除其中`spring.cloud.config`的内容。

同时在pom.xml文件中去除对应的依赖：
```xml
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-config</artifactId>
        </dependency>
```

再次本地构建运行。

发现程序可以启动成功，但是报错无法注册到Eureka Server。

## 去除注册到Eureka Server

修改Spring Boot应用类GatewayApplication.java，将`@EnableDiscoveryClient`注解去掉，并去掉对应的`import`语句。

同时在pom.xml文件中去除对应的依赖：
```xml
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
        </dependency>
```
或
```xml
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-eureka</artifactId>
        </dependency>
```

再次本地构建运行。

发现程序可以启动成功，但是报错无法连接到数据库（本例子数据库为MongoDB）。

在解决连接到MongoDB数据库问题之前，先检查一下剩余的Spring Cloud依赖。

## 剩余的Spring Cloud依赖

保留本服务中使用的Spring Cloud依赖：
- `spring-cloud-starter`
- `spring-cloud-starter-oauth2`
- `spring-cloud-starter-sleuth`

说明
- `spring-cloud-starter`依赖包含了`spring-cloud-starter-config`、`spring-cloud-starter-netflix-eureka-client`、`spring-cloud-starter-netflix-hystrix`、`spring-cloud-starter-netflix-ribbon`、`spring-cloud-starter-netflix-zuul`、`spring-cloud-starter-oauth2`、`spring-cloud-starter-sleuth`等依赖。


如果程序不再依赖任何Sprng Cloud组建，则可以去除全部Spring Cloud依赖，并再次本地构建运行。

## 应用配置

将原来的Spring Cloud Config Server中的gateway配置，放到application.yaml中。

并进行修改后如下：

```yaml
hystrix:
  command:
    default:
      execution:
        isolation:
          thread:
            timeoutInMilliseconds: 20000

ribbon:
  ReadTimeout: 20000
  ConnectTimeout: 20000

zuul:
  ignoredServices: '*'
  host:
    connect-timeout-millis: 20000
    socket-timeout-millis: 20000
  routes:
    auth-service:
        path: /uaa/**
        url: http://auth-service:8080
        stripPrefix: false
        sensitiveHeaders:
    account-service:
        path: /accounts/**
        serviceId: account-service
        stripPrefix: false
        sensitiveHeaders:
    statistics-service:
        path: /statistics/**
        serviceId: statistics-service
        stripPrefix: false
        sensitiveHeaders:
    notification-service:
        path: /notifications/**
        serviceId: notification-service
        stripPrefix: false
        sensitiveHeaders:
```

注意：
- 使用默认端口（8080）
- auth-service.url改为http://auth-service:8080

再次本地构建运行。

发现程序可以启动成功，但是这时配置认证服务连接还是有问题的：
- `http://auth-service:8080`其实在本地是访问不通的

## 本地调试运行

添加一个application-local.yaml文件，用于本地调试。

```yaml
server:
  port: 4000

zuul:
  routes:
    auth-service:
        url: http://localhost:5000
```

说明：
- 本地调试运行时，使用的端口为4000
- auth-service.url改为本地调试时的端口

本地调试运行：
```bash
mvn spring-boot:run -Dspring.profiles.active=local
```

## 访问本地Web页面

在浏览器中直接访问http://localhost:4000/ 来访问Web页面。

1. 点击CREATE NEW ACCOUNT
2. 输入用户名和密码
3. 点击REGISTER

发现报错，错误信息为Load balancer does not have available server for client: account-service。

这是因为我们已经去掉了对Eureka Server的依赖，所以Zuul无法通过Eureka Server来获取account-service的地址。

## 直接转发请求到服务

修改application.yml，将原来根据serviceId的转发改为根据url的转发。

代码片段：

```yaml
zuul:
  routes:
    auth-service:
        url: http://auth-service:8080
    account-service:
        url: http://account-service:8080
    statistics-service:
        url: http://statistics-service:8080
    notification-service:
        url: http://notification-service:8080
```

类似地，修改application-local.yml：

```yaml
zuul:
  routes:
    auth-service:
        url: http://localhost:5000
    account-service:
        url: http://localhost:6000
    statistics-service:
        url: http://localhost:7000
    notification-service:
        url: http://localhost:8000
```

再次本地调试运行。




## 构建容器镜像

创建Dockerfile文件：

```bash
FROM registry.access.redhat.com/ubi8/openjdk-8

COPY target/*.jar /opt/app.jar

CMD java -jar /opt/app.jar

EXPOSE 8080
```

先构建应用：
```bash
mvn clean install
```

构建容器镜像：

```bash
podman build -t gateway .
```

推送容器镜像到镜像仓库：

```bash
podman login quay.io
# replace as your repository
podman tag gateway:latest quay.io/xxx/gateway:latest
podman push quay.io/xxx/gateway:latest
```

## 部署到Kubernetes

在piggymeticrs-config仓库中集中管理多个微服务的Kubernetes YAML。



## Troubleshooting

### launch.js无法访问api.exchangeratesapi.io

修改launch.js直接harcode汇率数据。

```js
	global.eur = 1 / 0.011866105;
	global.usd = 1 / 0.012899035;
```

TODO 只有在Safari浏览器上输入用户名并按回车后，才能正常显示输入密码的框。




