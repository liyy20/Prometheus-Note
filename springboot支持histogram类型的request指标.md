# 背景
用户除了关注指标http_server_requests_seconds， 希望进一步关注TP90,TP99之列的分段指标。<br>
比如： 用户希望有summary计算90%的请求耗时情况； 希望排除几次请求偏大的数据；

`TP90就是满足90%的网络请求所需要的最低耗时(90%的请求耗时情况)。
TP99就是满足99%的网络请求所需要的最低耗时`

## 参考：
[● TP90,TP99,TP999,MAX含义](http://t.zoukankan.com/Tanwheey-p-12401485.html)<br>
[● prometheus的summary和histogram指标](https://blog.csdn.net/wtan825/article/details/94616813)
# 如何实现？ 
## 参考：
[spring-doc 6.5.2. Per-meter Properties](https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#appendix.dependency-versions.coordinates)
[Spring Boot 集成 Prometheus](https://blog.51cto.com/u_1472521/5050197)
## 1.前提
应用已经接入prometheus ， 参考： [java项目接入Prometheus监控]()
## 2.修改配置文件application.yaml，添加：
```YAML
management:
  metrics.distribution.percentiles-histogram.http.server.requests: true
  # Micormeter bucket指标配置，千分尺分段记录
  metrics.distribution.sla.http.server.requests: 100ms,200ms,400ms
  # Micormeter quantile 指标配置
  metrics.distribution.percentiles.http.server.requests: 0.5,0.9,0.95,0.99,0.999
```
## 3.启动应用
## 4.请求url: 
```Bash
curl  -s localhost:8080/actuator/prometheus  (8080为management  port）
```
## 5.输出：
```JavaScript
# HELP http_server_requests_seconds  
# TYPE http_server_requests_seconds histogram
http_server_requests_seconds{exception="None",method="GET",outcome="SUCCESS",status="200",uri="/actuator/prometheus",quantile="0.5",} 0.00524288
http_server_requests_seconds{exception="None",method="GET",outcome="SUCCESS",status="200",uri="/actuator/prometheus",quantile="0.9",} 0.043778048
http_server_requests_seconds{exception="None",method="GET",outcome="SUCCESS",status="200",uri="/actuator/prometheus",quantile="0.95",} 0.043778048
http_server_requests_seconds{exception="None",method="GET",outcome="SUCCESS",status="200",uri="/actuator/prometheus",quantile="0.99",} 0.043778048
http_server_requests_seconds{exception="None",method="GET",outcome="SUCCESS",status="200",uri="/actuator/prometheus",quantile="0.999",} 0.043778048
http_server_requests_seconds_bucket{exception="None",method="GET",outcome="SUCCESS",status="200",uri="/actuator/prometheus",le="0.001",} 0.0
http_server_requests_seconds_bucket{exception="None",method="GET",outcome="SUCCESS",status="200",uri="/actuator/prometheus",le="0.001048576",} 0.0
http_server_requests_seconds_bucket{exception="None",method="GET",outcome="SUCCESS",status="200",uri="/actuator/prometheus",le="0.001398101",} 0.0
http_server_requests_seconds_bucket{exception="None",method="GET",outcome="SUCCESS",status="200",uri="/actuator/prometheus",le="0.001747626",} 0.0
http_server_requests_seconds_bucket{exception="None",method="GET",outcome="SUCCESS",status="200",uri="/actuator/prometheus",le="0.002097151",} 0.0
http_server_requests_seconds_bucket{exception="None",method="GET",outcome="SUCCESS",status="200",uri="/actuator/prometheus",le="0.002446676",} 0.0
http_server_requests_seconds_bucket{exception="None",method="GET",outcome="SUCCESS",status="200",uri="/actuator/prometheus",le="0.002796201",} 0.0
http_server_requests_seconds_bucket{exception="None",method="GET",outcome="SUCCESS",status="200",uri="/actuator/prometheus",le="0.003145726",} 0.0
http_server_requests_seconds_bucket{exception="None",method="GET",outcome="SUCCESS",status="200",uri="/actuator/prometheus",le="0.003495251",} 0.0
```
