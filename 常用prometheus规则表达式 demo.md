# 1. 背景
各项目接入监控告警后，一个普遍的问题是prometheus规则表达式的书写。对于通用的告警规则，如web、jvm等，我们提供模版根据应用元数据来自动生成告警规则，避免了写promQL的工作。但是自定义业务指标则需要手写promQL，这里收集整理目前各业务线用到的告警规则作为参考demo。
有关于promQL的基本语法、常用内置函数，参考资料链接如下：<br>
[prometheus官方文档](https://prometheus.io/docs/introduction/overview/)<br>
[prometheus中文文档](https://prometheus.fuckcloudnative.io/)<br>
[prometheus内置函数(官方)](https://prometheus.io/docs/prometheus/latest/querying/functions/)<br>
[prometheus内置函数（中文）](https://prometheus.fuckcloudnative.io/di-san-zhang-prometheus/di-4-jie-cha-xun/functions)<br>

# 2. demo
指标中约定的通用tag如下。
tag|含义
----|----
app_kubernetes_io_group：| 应用所属组的名称
app_kubernetes_io_instance或app_kubernetes_io_name：|应用名称
app_kubernetes_io_envType：|应用部署标准环境名称，目前仅限于pre、test、prod三种
app_kubernetes_io_env：|应用部署子环境名称，自定义
pod：|应用部署实例名称，自动生成，每次部署改名称会变化
container：|容器类型，比如app、envoy-sidecar，通常我们只考虑pod的app类型容器
namespace：|kubernetes集群的命名空间，通常就是app_kubernetes_io_group

## A. 网络相关指标
`sum by (pod, uri, status)(increase (erpx_business_response_status_error_total{app_kubernetes_io_env="prod", app_kubernetes_io_group="erpx", app_kubernetes_io_instance="erpx-gateway", status =~ "1104|1103|1106|1107|"}[1m])) > 50`

含义：erpx组erpx-gateway服务在prod环境的http响应错误状态码指标，status表示自定义响应状态码，这里计算1104、1103、1106、1107四个状态码对应指标在1min内的增加量，按pod、uri、status、msg四个标签聚合，即该服务一个实例的一个接口上述四个状态status的响应数目1min内超过50作为告警阈值

## B. 容器重启、OOM相关指标
`increase(kube_pod_container_status_restarts_total{container="app",namespace="pdk"}[1m]) + on(namespace, pod) group_left(app_kubernetes_io_instance, app_kubernetes_io_envType, app_kubernetes_io_env) 0 * up{app_kubernetes_io_envType="prod",app_kubernetes_io_group="pdk",app_kubernetes_io_instance="pdk-mp-background"} > 0`

含义：kube_pod_container_status_restarts_total表示容器重启次数，pdk组pdk-mp-background服务生产环境容器重启告警，1min内该服务的实例容器重启次数 > 0作为告警阈值。这里通过namespace和pod两个标签的取值，将up指标中的应用名称、环境、子环境等标签关联到kube_pod_container_status_restarts_total重启指标，类似于mysql的关联查询。关联后的容器重启指标携带应用所属组、名称、环境信息，给出的告警报文更具有指导意义。
如果发现指标中缺乏需要的标签信息，可以利用group_left、group_right将更多的标签信息关联到目标指标

`sum by(app_kubernetes_io_instance, app_kubernetes_io_env) (kube_pod_container_status_last_terminated_reason{container="app",namespace="platform",reason="OOMKilled"} + on(namespace, pod) group_left(app_kubernetes_io_instance, app_kubernetes_io_env) 0 * up{app_kubernetes_io_env="test",app_kubernetes_io_group="platform",app_kubernetes_io_instance="platform-waybill-service"}) > 0`

含义：这里通过namespace、pod两个标签取值，将up指标中的应用所属组、子环境、应用名称关联到kube_pod_container_status_last_terminated_reason容器异常终止指标指标，容器异常终止的原因限定为OOM Killed，按子环境、应用两个标签聚合，即test子环境下platform-waybill-service服务发生oom即触发告警

## C. 业务相关指标
`increase(platform_exception_metric_total{app_kubernetes_io_env!="dy-other",app_kubernetes_io_env=~".*-other",module_name="logistic_method"}[1m]) > 0`

含义：platform组自定义的业务异常数目指标，过滤子环境dy-other，正则匹配*-other子环境，限定为logistic_method模块，该指标1min内增加量 > 0作为告警条件

`increase(platform_exception_metric_total{app_kubernetes_io_env=~"default-other",module_name="ksxd_http_client"}[1m]) > 0`

含义：platform组自定义的业务异常数目指标，子环境限定为default-other，模块名称限定为ksxd_http_client，该指标1min内增加量 > 0作为告警条件

`sum by(topic) (rocketmq_producer_offset) - on(topic) group_right() sum by(topic) (rocketmq_consumer_offset{topic="GYL_MIDDLEWARE_DISTRIBUTOR_FLINK_RECIVE_MERCHANT_AUTH"}) > 10000`

含义：rocketmq消息队列在一个topic上生产者生产数目与消费者消费数目之差，这里topic设置为"GYL_MIDDLEWARE_DISTRIBUTOR_FLINK_RECIVE_MERCHANT_AUTH"，按topic标签聚合并通过group_right关联

`avg(tm_mrds_queue_block_size{app_kubernetes_io_group="tm",app_kubernetes_io_instance="tm-mrds-reciever",platform="pdd",product=~"erp|ekb"}) > 10000`

含义：tm-mrds-reciever各个实例的消息队列大小平均值 > 10000

`(sum by(app_kubernetes_io_instance) 		(increase(platform_decrypt_order_failed_total{app_kubernetes_io_env="prod",app_kubernetes_io_group="tm",app_kubernetes_io_instance=~"platform-decrypt"}[1m])) 
/ 
sum by(app_kubernetes_io_instance) (increase(platform_decrypt_order_total{app_kubernetes_io_env="prod",app_kubernetes_io_group="tm",app_kubernetes_io_instance=~"platform-decrypt"}[1m]))) * 100 > 20`

含义：1min内服务的解密失败订单总量与解密订单总量之比 > 20%
分子表示platform-decrypt服务1min内的解密失败订单总量，分子表示1min的服务的解密订单总量，分子分母均按应用粒度做了聚合处理。

## D. 线程状态相关指标

`sum by(app_kubernetes_io_env, pod) (jvm_threads_states_threads{app_kubernetes_io_env=~"dy-other|pdd-other|ks-other|other-default",app_kubernetes_io_group="platform",app_kubernetes_io_instance=~"platform-sync-app",state=~"blocked"}) > 100`

含义：对于应用platform-sync-app在dy-other、pdd-other、ks-other、other-default子环境的每个实例，JVM进程处于blocked状态的线程数目 > 100作为告警条件

`tomcat_threads_busy_threads{app_kubernetes_io_group="platform",app_kubernetes_io_env !~"prod", app_kubernetes_io_instance=~"platform-waybill-mixer"}
/
tomcat_threads_config_max_threads{app_kubernetes_io_group="platform",app_kubernetes_io_env !~"prod", app_kubernetes_io_instance=~"platform-waybill-mixer"}*100 > 7`

含义：非生产环境platform-waybill-mixer服务tomcat忙碌线程数目与最大线程数目之比 > 7%作为告警条件
