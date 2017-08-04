# 如何使用

linkerd 作为单独的独立代理运行，释放其语言和类库要求。应用程序通常通过在已知位置运行 linkerd 实例来使用 linkerd，并通过这些实例代理调用，而不是直接连接到目标，服务连接到它们对应的 linkerd 实例，并将这些实例视为目标服务。

在私底下，linkerd 应用路由规则，与现有服务发现机制进行通信，以及目标实例之上的负载平衡 - 所有这些都是在调整通信和报告指标的过程中。 典型的 linkerd 设置如下所示：

![](images/diagram-individual-instance.png)

**Linkerd如何集成到现有应用**

By deferring the mechanics of making the call to linkerd, application code is decoupled from:

knowledge of the production topology;
knowledge of the service discovery mechanism; and
load balancing and connection management logic.

通过依从给 linkerd 发起调用的机制，应用程序代码可以解耦于：

- 产品拓扑知识
- 服务发现机制的知识
- 和负载平衡和连接管理逻辑

应用程序也受益于一致的全局流量控制机制。这对于多语言应用尤为重要，因为他们很难通过类库来实现这种一致性。

linkerd 实例可以部署为 sidecar（即每个应用服务实例一个实例）或每个主机。由于 linkerd 实例是无状态和独立的，因此它们可以轻易适应现有的部署拓扑。它们可以以各种配置的方式和应用程序代码一起部署，而只需要最少的协调。
