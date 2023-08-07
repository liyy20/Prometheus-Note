# 1. 概要
本文对SpringBoot项目中暴露的通用prometheus指标进行归纳整理，并解释这些指标的详细含义，这些指标在技术中台监控 & 报警系统中广泛使用，也可以作为使用监控报警系统的补充参考文档。
## 1.1 prometheus指标的格式和类型
### 1.1.1 prometheus指标的格式
prometheus指标的通用格式由指标名称和一组标签构成，如下所示：
metric_name{labe1Name="label1Value",label2Name="label2Value",...}
一个示例如下：
```JavaScript
http_server_requests_seconds_count{app_kubernetes_io_env="prod", app_kubernetes_io_group="tm", app_kubernetes_io_instance="tm-alarm", app_kubernetes_io_name="tm-alarm", app_kubernetes_io_type="spring", container="app", endpoint="mgr-http", exception="None", instance="10.4.3.119:9090", jck_appEnvId="38719", jck_appId="34604", job="monitoring/platform-spring-cicd", method="GET", namespace="tm", outcome="SUCCESS", pod="jck-deployment-yacs-34604-38719-1521477-65cf4c4449-7kpjl", prometheus="monitoring/k8s", receive_cluster="wdt", status="200", tenant_id="base", uri="/api/tm/alarm/2.0/prometheus/rule/query"}
```
其中http_server_requests_seconds_count是指标名称，后面花括号内是一组标签名-标签值对，利用标签可以进行多维度的统计和查询。这里我们对指标的一些常用标签作出说明。标签app_kubernetes_io_env表示应用所处的环境，对应值包括生产环境prod和测试环境test等。app_kubernetes_io_group和namespace分别表示应用所属组和命名空间，这两个标签取值相同。app_kubernetes_io_name和app_kubernetes_io_instance表示应用名称。通过app_kubernetes_io_group、app_kubernetes_io_instance、app_kubernetes_io_group这三个标签可以将指标的粒度确定到应用级别；进一步通过pod标签可以将指标的粒度细化到实例级别（这里的pod或实例概念同kubernetes一致）。
## 1.2 prometheus指标的类型
Prometheus定义了4中不同的指标类型(metric type)：Counter（计数器）、Gauge（仪表盘）、Histogram（直方图）、Summary（摘要）。Counter类型指标的值只增不减，除非发生重置（应用发布、重启等），该类型指标的值随着时间增长不断累积，比如应用HTTP请求数量、GC次数。Gauge类型的指标侧重于反映应用的当前状态，因此这类指标的值随着应用状态变化而改变，如应用CPU使用率、1分钟内耗时最长请求的时间、JVM使用的内存大小。
Summary类型的指标主要用于统计分析，反映指标值的分布状况，我们举例说明之。一般情况下对指标的统计分析主要采用平均值，比如HTTP请求响应时长（延迟），我们常用的计算方式为：1分钟内的请求总数 / 1分钟内的请求总耗时，这种平均值可能会掩盖指标值的异常状况，比如大部分请求耗时在毫秒左右，而个别请求耗时达到10s，假设1分钟内请求总数为100，那么请求平均响应时长约为100毫秒，仅仅只看指标平均值似乎响应时间在正常范围内。更好的分析方式是考虑响应时长的分布状况，将指标样本值按非递减序排序后，考虑前50%的响应时长、前90%的响应时长、前99%的响应时长。Summary类型的指标正反映了样本值的分布情况，这类指标中的quantile标签对应样本值的百分位数，典型的取值有0.5、0.9、0.95、0.99等，可以这样理解指标不同分位数标签对应值的含义：样本总数为N，分别对应将样本以非递减序排序后，最小样本位置为1，依次类推，最大位置为N，分别位于N * 0.5、 N * 0.9、N * 0.95、N * 0.99位置处样本的值，这分别等价于50%、90%、95%、99%的样本小于分位数处指标的值。<br>
Histogram类型指标直接反映了在不同区间内样本的个数，类似于直方图，区间通过标签le进行定义。

# 2. 各类指标的解释
SpringBoot项目暴露的通用指标大致可以分为以下几类：HTTP请求、JVM、系统 & 进程、数据库连接、tomcat、日志。其中关于监控服务状态最核心的是HTTP请求和JVM进程相关指标。下面我们分类别介绍这些指标的含义。
## 2.1 HTTP请求相关指标
SpringBoot暴露的默认HTTP请求指标用于监控Web应用QPS（流量）、延迟（响应时间）、错误率。
指标|含义
---- | ----
http_server_requests_seconds_count: |http所有请求总数，计数器类型。
http_server_requests_seconds_sum: |http所有请求总耗时，单位为秒，计数器类型。重要标签与http请求总数指标一致，同理，通过uri标签可以确定每个接口所有请求的总耗时。
http_server_requests_seconds_max: |在时间窗口内的耗时最长请求的持续时间，单位为秒，仪表盘类型。当新的时间窗口开始时，该值将重置为0，默认时间窗口为2分钟。
http_server_requests_seconds: |http请求耗时时长分布，单位为秒，Summary类型。

上述四个指标中重要的标签有uri、method、status、exception，分别对应HTTP请求接口路径、方法、状态码、抛出的异常类名称。通过uri标签可以明确具体接口的请求总数、所有请求的总耗时、在时间窗口内该接口请求最长耗时，如图1所示。

通过指标http_server_requests_seconds分位数标签quantile我们可以进一步知道在一段时间内耗时在前50%、前90%、前95%、前99%的请求时长变化曲线，如图2所示

## 2.2 JVM相关指标
SpringBoot中提供的JVM指标分为三类：JVM 内存、JVM GC、JVM 线程，分别提供了SpringBoot应用JVM内存使用情况、GC状态、线程状态的监控信息。
### 2.2.1 JVM内存
指标|含义
---- | ----
jvm_memory_max_bytes:  |JVM内存管理所能使用的最大内存，单位为字节
jvm_memory_committed_bytes: |分配给JVM的可用内存，单位为字节
jvm_memory_used_bytes: |JVM已使用的内存大小，单位为字节

上述三个指标的关系为：jvm_memory_max_bytes >= jvm_memory_committed_bytes >= jvm_memory_used_bytes，即最大内存不低于分配内存，分配内存不低于已使用内存。对于JVM内存，我们通常粗略地将其划分为堆内存和非堆内存，上述三个指标的area标签对应这种内存区域划分：area = "heap"代表堆区，area = "nonheap"代表非堆区。在实践中我们通常重点关注JVM堆区的内存使用情况，这可以用堆区使用内存与最大内存之比来刻画，即堆内存使用率。内存区域的进一步细化可以通过上述三个指标的id标签来实现，若标签area = "heap"，标签id可取值为：Eden Space、Survivor Space、Tenured Gen，分别对应堆内存的各代大小；若area = "nonheap'，标签id可取值为：Metaspace、Compressed Class Space、CodeHeap 'profiled nmethods'、CodeHeap 'non-profiled nmethods'、CodeHeap 'non-nmethods'。通过area和id标签可以将JVM内存监控指标细化到各区域。
指标|含义
---- | ----
jvm_buffer_total_capacity_bytes: |JVM缓冲区的总容量，单位为字节
jvm_buffer_memory_used_bytes: |JVM缓冲区的已使用的内存大小，单位为字节
jvm_buffer_count_buffers: |JVM缓冲区的数量

上述JVM缓冲区三个指标中最重要的标签是id，取值为direct或map，分别对应JVM中直接缓冲区和非直接缓冲区
指标|含义
---- | ----
jvm_classes_loaded_classes: |JVM当前加载的类的数量 ，仪表盘类型
jvm_classes_unloaded_classes_total: |JVM自运行起卸载的类的总数量，计数器类型
### 2.2.2 JVM GC
指标|含义
---- | ----
jvm_gc_pause_seconds_count: |JVM GC次数，计数器类型
jvm_gc_pause_seconds_sum: |JVM GC耗时，单位为秒，计数器类型
jvm_gc_pause_seconds_max：|JVM GC最大耗时时长，单位为秒，仪表盘类型

上述三个指标提供了JVM GC时间花销方面的信息，通常我们关注GC的动作action和原因cause，这三个指标的action和cause标签提供了GC时间消耗进一步信息，标签action的取值有：end of minor GC、end of major GC，分别对应年轻代空间（包括 Eden 和 Survivor 区域）、老年代的内存回收，cause的取值有：Allocation Failure、Metadata GC Threshold、G1 Evacuation Pause、G1 Humongous Allocation、GCLocker Initiated GC。
指标|含义
---- | ----
jvm_gc_memory_allocated_bytes_total: |JVM上一个 GC 之后堆区年轻代内存池的大小增加
jvm_gc_memory_promoted_bytes_total: |JVM上一个 GC之后的老年代内存池大小的增加量
jvm_gc_max_data_size_bytes: |老年代内存池的最大大小
jvm_gc_live_data_size_bytes: |Full GC 后的老年代内存池的大小
### 2.2.3 JVM线程
指标|含义
---- | ----
jvm_threads_states_threads|
jvm_threads_peak_threads: |自 Java 虚拟机启动或峰值重置以来的最高活动线程数
jvm_threads_daemon_threads: |JVM当前守护线程数量 
jvm_threads_live_threads: |JVM当前线程数，包括守护进程和非守护进程的线程
## 2.3 系统 & JVM CPU和文件相关指标
指标|含义
---- | ----
up: |可用性指标，该指标值为1时代表正常，为0时代表不可用。
system_cpu_count: |运行JVM进程的CPU核数，底层通过方法java.lang.Runtime.availableProcessors()获取
process_cpu_usage: |JVM进程CPU使用率，仪表盘类型。这里我们需要了解CPU使用率的计算方式，这里是JVM进程在一段时间内占用的CPU时间与总的CPU时间的百分比。在处理器为多核架构且进程采用多线程模型的情况下，CPU使用率是可以超过100%的，比如某个开启多线程的JVM进程1s内占用了CPU0 0.6s, CPU1 0.9s, 那么它的占用率是150%。这样就不难理解JVM进程CPU使用率指标超过100%的情况了。这种计算CPU使用率的方式与我们在top命令中看到的CPU使用率计算方式一致。
system_cpu_usage: |宿主机CPU使用率 ，仪表盘类型。这里的计算方式是将一段时间内操作系统的用户态时间、内核态时间、空闲状态时间三者之和除以总时间，即运行虚拟机 CPU 时间、等待 I/O 的 CPU 时间对宿主机CPU使用率没有贡献。
system_load_average_1m: |在1min内，令T为等待CPU的可调度任务数量与在CPU上运行任务数量之和，令N为CPU核数，在当前1min内CPU负载平均值为 T / N。
process_uptime_seconds: |JVM进程的正常运行时间，单位为秒
process_start_time_seconds: |JVM进程启动时间
process_files_open_files: |JVM进程打开文件描述符的数量
process_files_max_files: |JVM进程最大文件处理数量，默认为1024
## 2.4 tomcat相关指标
指标|含义
---- | ----
tomcat_threads_config_max_threads:| Tomcat配置的最大线程数 
tomcat_threads_busy_threads:|  Tomcat繁忙线程数
tomcat_threads_current_threads:| Tomcat当前的线程数
tomcat_connections_keepalive_current_connections|
tomcat_connections_config_max_connections|
tomcat_connections_current_connections|
tomcat_threads_config_max_threads|
tomcat_sessions_active_current_sessions:| Tomcat当前活跃 session 数量
tomcat_sessions_created_sessions_total:| Tomcat创建的session 数
tomcat_sessions_expired_sessions_total:| Tomcat过期的 session 数量
tomcat_sessions_alive_max_seconds:| Tomcat session 最大存活时间 
tomcat_sessions_active_max_sessions|
tomcat_sessions_rejected_sessions_total:| Tomcat被拒绝的 session 总数
tomcat_cache_hit_total|
tomcat_cache_access_total|
tomcat_global_request_seconds_count:| Tomcat全局请求总数
tomcat_global_request_seconds_sum:| Tomcat 全局请求总耗时
tomcat_global_request_max_seconds:| Tomcat全局最长请求的耗时
tomcat_global_received_bytes_total:| Tomcat接收到的数据量 
tomcat_global_sent_bytes_total:| Tomcat发送的数据量 
tomcat_global_error_total:| Tomcat 全局异常数量
tomcat_servlet_error_total|
tomcat_servlet_request_max_seconds|
tomcat_servlet_request_seconds_count|
tomcat_servlet_request_seconds_sum|

## 2.5 数据库连接相关指标
todo
指标|含义
---- | ----
jdbc_connections_max|
jdbc_connections_min|
jdbc_connections_idle|
jdbc_connections_active|
hikaricp_connections_min|
hikaricp_connections_idle|
hikaricp_connections_active|
hikaricp_connections_max|
hikaricp_connections|
hikaricp_connections_pending|
hikaricp_connections_usage_seconds_count|
hikaricp_connections_usage_seconds_sum|
hikaricp_connections_usage_seconds_max|
hikaricp_connections_timeout_total|
hikaricp_connections_creation_seconds_count|
hikaricp_connections_creation_seconds_sum|
hikaricp_connections_creation_seconds_max|
hikaricp_connections_acquire_seconds_count|
hikaricp_connections_acquire_seconds_sum|
hikaricp_connections_acquire_seconds_max|
## 2.6 日志相关指标
指标|含义
---- | ----
logback_events_total:| 日志按级别的计数数量，日志级别通过标签level确定，一共有debug、info、warn、error、fatal五个级别
# 3. 自定义指标demo
