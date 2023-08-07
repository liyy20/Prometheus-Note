# 一、概要
本文介绍java项目如何集成Prometheus监控、暴露监控指标，以及grafana可视化展示。默认项目采用SpringBoot框架，通过Maven管理依赖包。
# 二、暴露监控指标
通过SpringBoot Actuator 模块采集应用内部信息暴露给prometheus监控系统。Micrometer 为 Java 应用收集性能指标，并完成与监控系统的适配工作。Spring Boot Actuator 模块提供了所谓的端点（ endpoints ）给外部来与应用程序进行访问和交互，从应用性能监控的角度而言，主要关注以下三个端点
health端点聚合了应用的健康指标，来检查程序的健康情况
metrics端点用来返回当前应用的各类重要度量指标，比如：内存信息、线程信息、垃圾回收信息、tomcat、数据库连接池等
prometheus端点暴露了被Prometheus服务器抓取的指标数据

引入 spring-boot-starter-actuator依赖
```xml
<dependency>
<groupId>org.springframework.boot</groupId>
<artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```
引入Micrometer依赖
```xml
<dependency> 
<groupId>io.micrometer</groupId> 
<artifactId>micrometer-registry-prometheus</artifactId> 
</dependency>
```
注：无特别需求，不需要增加版本号。
修改项目配置文件application.yaml
暴露metric地址给prometheus（必选）
在management.endpoints.web.exposure.include 配置项中加入prometheus, metrics两项(其他include选项自己决定是否添加）。prometheus endpoints
yaml格式示例如下
```YMAL
management:
  endpoints:
    web:
      exposure:
        include:
          - health
          - info
          - metrics
          - env
          - beans
          - conditions
          - loggers
          - mappings
          - prometheus
```
暴露tomcat监控指标 （必选）
```properties
server.tomcat.mbeanregistry.enabled = true。
```
yaml格式示例如下
```YAML
server:
  tomcat:
    mbeanregistry:
      enabled: true
```
添加自定义tag（可选）
```properties
management.metrics.tags = ....
```
yaml格式示例如下
```YAML
management: 
  metrics:
    tags:
      application: ${spring.application.name}
```
 添加自定义指标（可选）
参考自定义指标
检查指标是否暴露成功
启动SpringBoot服务（默认服务端口是9090），在终端中输入以下命令查看应用暴露的监控指标：
A 聚石塔容器内
prometheus metric格式：
```Bash
curl -i -X GET  'http://localhost:9090/actuator/prometheus'
```
返回如下图所示

所有metric名称：
```Bash
curl -i -X GET  'http://localhost:9090/actuator/metrics'
```
返回如下图所示

B 本地开发环境
prometheus metric格式：
```Bash
curl -i -X GET  'http://localhost:8080/actuator/prometheus'
```
所有metric名称：
```Bash
curl -i -X GET  'http://localhost:8080/actuator/metrics'
```
# 三、常用组件的监控
1. open feign客户端
请按照技术中台的规范映入open feign相关依赖，在上述监控配置的基础上，额外映入如下依赖包即可
相关指标请见grafana, Java App文件夹下的web指标，出站请求（open feign客户端）
2. dynamic tp动态线程池
这里无需引入额外的配置和依赖，按照官方文档在pom文件中加入下面的依赖后直接可使用dynamic tp，监控指标自动暴露
四、grafana监控大盘的使用
各云的监控大盘地址
A. 掌上先机聚石塔(jst ERP)监控
B. 腾讯云监控
C. 跨境聚石塔
指标查询
进入grafana https://grafana.huice.com
选择 ·explore·
选择数据源 `thanos`
输入指标 ·topk(10,tomcat_connections_current_connections) ·， 查看结果：


