# 部署

对于 linkerd , 有两个常见的部署模型：每主机(per-host)和作为边车(sidecar)进程。

## 每主机

在每主机部署模型中，每个主机（无论是物理机还是虚拟机）部署一个 linkerd 实例，然后该主机上的所有应用程序服务实例都通过此实例路由流量。

该模型对于主要基于主机的部署是很有用的。主机上的每个服务实例可以在固定位置（通常为localhost：4140）定位其对应的 linkerd 实例，从而避免了任何重要的客户端逻辑的要求。

由于该模型需要 linkerd 实例的高并发性，因此更大的资源配置文件通常是适当的。在此模型中，单个 linkerd 实例的丢失等同于丢失主机本身。

![](images/diagram-per-host-deployment.png)
**Linkerd每主机部署**

## 边车

在边车部署模型中，每个应用程序服务的实例都部署了一个 linkerd 实例。该模型对于主要是基于实例或基于容器的部署而言非常有用，而不是基于主机。例如，使用 Kubernetes 部署，可以将一个 linkerd 容器作为 Kubernetes “pod” 的一部分进行部署，并且服务实例可以定位 linkerd 实例,就好像在同一个主机上一般，如连接到 `localhost:4140`。

由于这种边车方式需要许多 linkerd 实例，所以较小的资源配置文件通常是合适的。在此模型中，单个 linkerd 实例的丢失等同于丢失相应的服务实例。

![](images/diagram-sidecar-deployment.png)
**Linkerd以边车进程部署(服务到linkerd)**

应用程序服务和 linkerd 如何相互通信有三种配置：服务到linkerd，linkerd到服务以及linkerd到linkerd。

### 服务到linkerd

在服务到 linkerd 配置中，每个服务实例通过其对应的 linkerd 实例来路由调用。每个 linkerd 将在匹配服务实例已知的位置提供服务，并将流量路由到远程服务。

### linkerd到服务

在 linkerd 到服务配置中，应用程序服务实例不直接服务于流量。相反，边车 linkerd 应该在服务发现中进行注册，以便传入流量由 linkerd 服务，然后将其路由到匹配的服务实例。虽然此配置丢失了 linkerd 的所有客户端优势，但它确实为您提供了应用程序服务的服务器metrics，例如请求计数和延迟直方图。

### linkerd到linkerd

linkerd到linkerd 配置是上述两种配置的组合，并为您提供了两个方面的最佳选择。边车 linkerd 应该在服务发现中注册，以便传入的流量由 linkerd 服务，然后 linkerd 将其路由到匹配的服务实例。然后服务实例通过 linkerd 路由外出的呼叫。这通常需要在 linkerd 配置中设置两个路由器：一个用于传入流量，一个用于传出流量。
