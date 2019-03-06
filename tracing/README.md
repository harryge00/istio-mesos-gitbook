### envoy支持trace
envoy原生就支持分布式追踪系统的接入，如支持jaeger和zipkin，如[envoy的Tracing官方文档](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/tracing#arch-overview-tracing)中表明envoy支持如下trace特性：

* 生成Request Id，填充HTTP的header字段x-request-id
* 外部跟踪服务集成，如支持LightStep, Zipkin或任何Zipkin兼容后端(如Jaeger)
* 添加Client trace ID

相关信息可以参考[envoy 中文文档](https://www.envoyproxy.cn/Buildingandinstallation/Sandboxes/JaegerTracing.html)，另外，在源码中有对应的[trace相关的源码](https://github.com/envoyproxy/envoy/blob/master/source/common/tracing/http_tracer_impl.cc)。
### Istio支持trace
Istio的分布式追踪相关介绍里面相关说明可以知道，Istio的envoy代理拦截流量后会主动上报trace系统，通过proxy的参数zipkinAddress指定了trace系统的地址，这样就不会再经过mixer了，直接envoy和trace系统交互，大体流程：

* 如果incoming的请求没有trace相关的headers，则会再流量进入pods之前创建一个root span
* 如果incoming的请求包含有trace相关的headers，Sidecar的proxy将会extract这些span的上下文信息，然后再在流量进入pods之前创建一个继承上一个span的新的span


由于Istio的proxy代理是envoy，而envoy又原生支持jaeger，那因此Istio自然而然就支持jaeger了，在官方文档[Distributed Tracing](https://istio.io/docs/tasks/telemetry/distributed-tracing/)中有相关较为详细的说明。

### 访问jaeger
首先获取jaeger的端口与IP，可以在marathon ui上获取，端口为`query-http` 即为UI端口。
![jager](https://istio.io/docs/tasks/telemetry/distributed-tracing/istio-tracing-list.png)
