本文我们介绍如何在项目中自定义prometheus业务监控指标。
# 1. 前提条件
## 1.1 接入prometheus配置 & 依赖
参考prometheus监控系统接入
## 1.2 背景
`2022-12-05修改：去掉HashMap缓存指标，在多线程场景下，使用HashMap缓存不安全。此外，MicroMeter依赖包本身已经对指标做了缓存，用的是ConcurrentHashMap，考虑了多线程安全。`
这里我们以订单处理服务为背景，提供自定义业务指标的demo代码。假设接口orderHandler接收来自不同平台platform和店铺shop的订单，调用processOrder方法处理订单，返回值-1表示订单处理失败，0表示处理成功，我们希望能定义关于不同平台和商户的接收订单总数、失败订单总数、订单处理总耗时三个业务指标。
接收订单的orderHandler接口代码如下
```java
package com.xxx.tc.demo.monitor;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;

/**
 * @author lyy
 */
@RestController
@RequestMapping("/api/tm-webapp-demo/order")
public class OrderController {

    private final OrderService orderService;

    @Autowired
    public OrderController(OrderService orderService) {
        this.orderService = orderService;
    }

    /**
     * @param platform 订单来自的平台
     * @param shop 订单来自的商户
     * @return 订单处理结果状态码，-1表示失败，0表示成功
     */
    @GetMapping("/handler")
    public int orderHandler(@RequestParam String platform, @RequestParam String shop) throws InterruptedException {
        return orderService.processOrder(platform, shop);
    }

}
```
处理订单的方法processOrder如下
```java
package com.xxx.tc.demo.monitor;

import org.springframework.stereotype.Service;
import java.util.Objects;

/**
 * @author chengxiyang
 */
@Service
public class OrderService {

    /**
     * mock订单处理服务
     * @param platform 平台
     * @param shop 商户
     * @return mock订单处理结果状态码
     */
    int processOrder(String platform, String shop) throws InterruptedException {
        if (Objects.isNull(platform) || Objects.isNull(shop)) {
            return -1;
        }

        Thread.sleep(100);

        if (platform.length() > shop.length()) {
            return 0;
        }
        else {
            return -1;
        }
    }
}
```
# 2. 配置自定义指标 
引入micrometer-registry-prometheus依赖包后，SpringBoot会自动注册MeterRegistry类型的bean用于上报自定义指标。在下面的demo代码中，我们配置了接收订单总数、处理失败订单总数、处理订单总耗时三个指标，名称分别为received_order、failed_order、order_process_duration，接收订单总数和处理失败订单总数两个指标包含两个tag：platform和shop，用以确定对应的平台和商户；处理订单总耗时指标只有platform这个tag，用以确定对应的平台。每个指标都有具体的描述信息，通过register(meterRegistry)方法上报给prometheus监控系统。我们通过platform和shop两个标签从map中取出对应的指标，通过increment()或record()方法打点。
`这里需要注意，指标中tag的取值一定是有限确定的，否则标签取值过于庞大会导致prometheus监控系统内存溢出。`<br>典型的例子是，tag的取值里面包含时间戳，或者用请求的traceId作为标签，由于时间戳和traceId的取值不是一个确定的有限集合，随着打点次数的累积指标数据量异常庞大，最终prometheus监控系统崩溃。
```java
package com.xxx.tc.demo.monitor;

import io.micrometer.core.instrument.Counter;
import io.micrometer.core.instrument.Gauge;
import io.micrometer.core.instrument.MeterRegistry;
import io.micrometer.core.instrument.Timer;
import io.micrometer.core.instrument.binder.BaseUnits;
import org.springframework.stereotype.Component;

import java.lang.management.ClassLoadingMXBean;
import java.lang.management.ManagementFactory;
import java.util.*;
import java.util.concurrent.TimeUnit;

/**
 * @author chengxiyang
 */
@Component
public class OrderMetric {

    private final MeterRegistry meterRegistry;

    private final OrderService orderService;

    /**
     * 接收订单总数指标的名称
     */
    private final static String RECEIVED_ORDER_COUNTER = "received_order";

    /**
     * 处理失败订单总数指标的名称
     */
    private final static String FAILED_ORDER_COUNTER = "failed_order";

    /**
     * 处理订单总耗时指标的名称
     */
    private final static String ORDER_PROCESS_DURATION = "order_process_duration";

    /**
     * 一分钟内的订单数
     */
    private final static String ORDER_PER_MIN = "order_per_min";


    /**
     * 指标的标签platform
     */
    private final static String PLATFORM = "platform";

    /**
     * 指标的标签shop
     */
    private final static String SHOP = "shop";

    public OrderMetric(MeterRegistry meterRegistry, OrderService orderService) {
        this.meterRegistry = meterRegistry;
        this.orderService = orderService;
    }

    /**
     * 配置接收订单总数指标，计数器类型，设置该指标的名称、标签名称和值、以及描述信息
     * @param platform 标签platform的值
     * @param shop 标签shop的值
     * @return 表示接口orderHandler(String platform, String shop)收到的订单总数
     */
    private Counter getReceivedOrderCounter(String platform, String shop) {

        Counter.Builder builder = Counter.builder(RECEIVED_ORDER_COUNTER)
                .tag(PLATFORM, platform)
                .tag(SHOP, shop)
                .description("total order received from specified platform and shop");
        return builder.register(meterRegistry);
    }

    /**
     * 配置处理失败订单总数指标，计数器类型，设置该指标的名称、标签名称和值、以及描述信息
     * @param platform 标签platform的值
     * @param shop 标签shop的值
     * @return 表示方法processOrder(String platform, String shop)处理失败的订单总数
     */
    private Counter getFailedOrderCounter(String platform, String shop) {

        Counter.Builder builder = Counter.builder(FAILED_ORDER_COUNTER)
                .tag(PLATFORM, platform)
                .tag(SHOP, shop)
                .description("total failed order of specified platform and shop");
        return builder.register(meterRegistry);
    }

    private Gauge getOrderPerMinGauge() {

        return Gauge.builder(ORDER_PER_MIN, orderService, OrderService::getOrderPerMin)
                .description("receiver order in one minute of specified platform and shop")
                .register(meterRegistry);
    }

    /**
     * 配置处理订单总耗时指标，Timer类型，设置该指标的名称、标签名称和值、以及描述信息
     * @param platform 标签platform的值
     * @return 表示方法processOrder(String platform, String shop)处理所有的订单总耗时
     */
    private Timer getOrderProcessingDuration(String platform) {
        Timer.Builder builder = Timer.builder(ORDER_PROCESS_DURATION)
                .tag(PLATFORM, platform)
                .description("total processing duration of order from specified platform");
        return builder.register(meterRegistry);
    }

    /**
     * 接收订单总数指标打点，表示该指标值加1
     * @param platform 标签platform的值
     * @param shop 标签shop的值
     */
    public void increaseReceivedOrder(String platform, String shop) {
        getReceivedOrderCounter(platform, shop).increment();
    }

    /**
     * 处理失败订单总数指标打点，表示该指标值加1
     * @param platform 标签platform的值
     * @param shop 标签shop的值
     */
    public void increaseFailedOrder(String platform, String shop) {
        getFailedOrderCounter(platform, shop).increment();
    }

    /**
     * 处理订单总耗时指标打点，表示耗时时长增加duration，时间单位为unit
     * @param platform 标签platform的值
     * @param duration 时长
     * @param unit 时长单位
     */
    public void recordOrderProcessDuration(String platform, long duration, TimeUnit unit) {
        getOrderProcessingDuration(platform).record(duration, unit);
    }


}
```

3. 指标的打点
指标的打点即对指标值进行修改，对于接收订单总数、处理失败订单总数、处理订单总耗时三个指标，我们希望在调用订单处理方法processOrder()时给platform和shop对应的订单数加1；处理完成后，检查processOrder()方法返回的状态码，如果为-1则给platform和shop对应的失败订单数加1；获取processOrder()方法执行前后的时间戳start和end，将end - start作为该次处理来自platform的订单的耗时并累加道耗时指标中。切面非常适合这种指标打点操作，示例代码如下
```java
package com.xxx.tc.demo.monitor;

import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.*;
import org.springframework.stereotype.Component;

import java.util.concurrent.TimeUnit;

/**
 * 利用切面在目标方法前后给自定义业务指标打点
 * @author chengxiyang
 */
@Aspect
@Component
public class MonitorAspect {

    private final OrderMetric orderMetric;

    public MonitorAspect(OrderMetric orderMetric) {
        this.orderMetric = orderMetric;
    }

    /**
     * 在com.huice.tc.demo.monitor.OrderService.processOrder(..)方法处给指标打点
     * @param point 切点
     * @return 返回值
     * @throws Throwable 异常
     */
    @Around(value = "execution (* com.huice.tc.demo.monitor.OrderService.processOrder(..))")
    public Object increaseCounter(ProceedingJoinPoint point) throws Throwable {

        String platform = (String) (point.getArgs())[0];
        String shop = (String) (point.getArgs())[1];

        long start = System.currentTimeMillis();

        int resultCode = (int) point.proceed();

        long end = System.currentTimeMillis();

        orderMetric.increaseReceivedOrder(platform, shop);
        if (resultCode == -1) {
            orderMetric.increaseFailedOrder(platform, shop);
        }
        orderMetric.recordOrderProcessDuration(platform, end - start, TimeUnit.MILLISECONDS);
        return  resultCode;
    }

}
```

# 4. metrics测试
这里我们可以通过向接口orderHandler发送请求模拟订单操作，比如platform取jingdong、taobao，shop取huice、shop_of_houlibin。发送若干次模拟请求后，在本地开发环境下发送如下请求查看暴露的监控指标
```Bash
curl -i -X GET \
 'http://localhost:8080/actuator/prometheus'
```
返回的结果中应该如下。

可以看到，计数器类型的指标在我们设定的名称后面附加了total，HELP行展示了指标的描述信息，TYPE行给出了指标的类型，每一行给出了指标在不同tag的当前值。

除了订单处理总耗时指标order_process_duration_seconds_sum之外，micrometer还额外生成了order_process_duration_seconds_count、order_process_duration_seconds_max两个指标，分别表示具体平台对应的订单总数、处理订单最大时长。
# 5.grafan查询自定义指标
grafana指标查询
# 6.创建自定义监控大盘 
[参考文档：](https://grafana.com/docs/grafana/latest/dashboards/dashboard-create/)
