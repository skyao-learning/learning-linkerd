# namerd

namerd 是为多个 linkerd 实例管理路由的服务。它通过存储 [dtabs](dtabs.md) 并使用 namers 进行服务发现来实现这个功能。namerd 支持与 linkerd 相同的服务发现后端套件，包括 ZooKeeper，Consul，Kubernetes API和 Marathon 等服务。

使用 namerd ，单独的 linkerd 不再需要直接与服务发现通话，或者将 dtabs 硬编码到其配置文件中。相反，他们向 namerd 询问所有必要的路由信息。这给我们带来了一些好处：

## 减少服务发现后端的负载

使用 namerd 意味着只有一小群 namerd 需要直接与服务发现后端通话，而不是队列中的每个linkerd。namerd 还利用缓存来进一步保护服务发现后端免受过载。

## 全局路由策略

通过将 dtabs 存储在 namerd 中，而不是硬编码在 linkerd 配置中，它可以确保路由策略在整个队列中同步，并在您需要进行更改时，为您提供一个真正的中心源。

## 动态路由策略

在 namerd 中存储 dtabs 的另一个优点是可以使用 namerd 的 API 或命令行工具动态更新这些 dtabs。这样可以执行诸如canary，staging或 blue-green 部署等操作，而无需重新启动任何 linkerd。

## 更多信息

要了解有关 namerd，搭建及其运维的更多信息，请查看 [Booyant的动态路由博客文章](https://blog.buoyant.io/2016/05/04/real-world-microservices-when-services-stop-playing-well-and-start-getting-real/index.html?__hstc=249056664.3c6b78fb9cb62c68eaaac6558454a06e.1501146055259.1501834234283.1501842962030.12&__hssc=249056664.1.1501842962030&__hsfp=4035021484#dynamic-routing-with-namerd)。

配置您自己的 namerd，转到 [namerd 配置文档](https://linkerd.io/config/1.1.2/namerd)。 另外还可以看看 [namerctl](https://github.com/linkerd/namerctl)，我们的控制 namerd 的开源工具。

为了一步一步的演练在 Kubernetes 中运行 namerd，以便持续部署，请查看 Buoyant 的博客文章，[通过流量转移持续部署](https://blog.buoyant.io/2016/11/04/a-service-mesh-for-kubernetes-part-iv-continuous-deployment-via-traffic-shifting/?__hstc=249056664.3c6b78fb9cb62c68eaaac6558454a06e.1501146055259.1501834234283.1501842962030.12&__hssc=249056664.1.1501842962030&__hsfp=4035021484)。


