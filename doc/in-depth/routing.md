# 路由

在其核心，linkerd 的主要工作是路由：接受请求(HTTP，Thrift，Mux或其他协议)，并将该请求发送到正确的目标。本指南将详细解释 linkerd 如何确定请求应该发送到哪里。这个过程由4个步骤组成：identification/识别，binding/绑定，resolution/解析和load balancing/负载均衡。

![LINKERD ROUTING](images/routing1.png)

**Linkerd 路由**

## 识别

Identification/识别 是将一个名称（也称为路径）分派给该请求的动作。名称是斜杠分隔的字符串，表示请求的目的地。默认情况下，linkerd 使用一个名为 `io.l5d.header.token` 的 identifier，该 identifier 根据 Host header 为请求命名，如下所示：`/svc/<HOST>`。这意味着 `GET http://example/hello` 的HTTP 请求将被分配名称 `/svc/example`。

（请注意，该URL的路径, `/hello`, 在名称中被删除。它仍将作为请求的一部分被代理转发 - 该名称仅决定请求如何路由，而不是发送到目标服务的内容）

当然，identifier 是一个可插拔的模块，并可以用自定义的 identifier 来替换，这个 identifier 基于你想要的任何逻辑来分配名称给请求。请在 [linkerd identifier 文档](https://linkerd.io/config/1.1.2/linkerd#http-1-1-identifiers) 中了解更多关于 linkerd 内置的 identifier 以及如何配置它们。

identifier 分配给请求的名称称为 service name，因为它应该编码应用程序指定的目标地址。它通常不编码群集，区域，环境或主机的信息，因为您的应用程序不需要担心这些问题。

例如，如果您的应用程序想要向 "users" 服务发出请求，它可以发出一个 HTTP GET 请求给 linkerd，以 "users" 作为 Host header。 `io.l5d.header.token` identifier将分配 `/svc/users` 作为该请求的 service name。

## 绑定

一旦将服务名称分配给请求，该名称将被 dtab（delegation table/委托表的简称）进行转换。这被称为 binding/绑定。有关 dtab 转换如何工作的详细文档可以在 [Dtabs页面](dtabs.md) 中找到。Dtabs 编码描述 service name 如何转换为 client name 的路由规则。client name 是 replica set/副本集的名称，通常是服务发现条目的名称。与service name 不同，client name 通常包含集群，区域和/或环境等细节。

client names 通常以 `/$` or `/#` 开头. (两个前缀的差别请看下面)

继续这个例子，假设我们有以下 dtab：

```bash
/env => /#/io.l5d.serversets/discovery
/svc => /env/prod
```

service name `/svc/users` 将像这样被绑定:

```bash
/svc/users
/env/prod/users
/#/io.l5d.serversets/discovery/prod/users
```

最终 `/#/io.l5d.serversets/discovery/prod/users` 成为 client name.

## 解析

解析/Resolution 是将 client name 解析为一组物理端点（IP地址+端口）。解析是通过称为 namer 的东西来完成的，通常会对某些服务发现后端进行查找。linkerd 带有大量内建的主流服务发现的实现。请在 [linkerd namer文档](https://linkerd.io/config/1.1.2/linkerd#namers) 中详细了解如何配置它们。

以 `/$` 开头的 client name 表示应该加载 classpath 中的 namer 来绑定该名称，而以 `/＃` 开头的 client name 表示加载 linkerd 配置中的 namer 来绑定该名称。

例如，假设我们有一个 client name `/#/io.l5d.serversets/discovery/prod/user`。这意味着来自 linkerd 配置的`io.l5d.serversets` namer 应该查找 `/discovery/prod/users` 服务器集（查找的结果是一组物理地址）。

类似地，client name `/$/inet/users/8888` 意味要为 inet namer 的搜索classpath。通过对 "users" 进行 DNS 查找并使用端口8888，该 namer 获取一组地址。

## 负载均衡

一旦 linkerd 拥有副本集，它使用 [负载平衡算法](https://blog.buoyant.io/2016/03/16/beyond-round-robin-load-balancing-for-latency/?__hstc=249056664.3c6b78fb9cb62c68eaaac6558454a06e.1501146055259.1501829257006.1501834234283.11&__hssc=249056664.4.1501834234283&__hsfp=4035021484) 来确定发送请求到哪里。因为 linkerd 在请求层而不是在连接层进行负载平衡，所以负载平衡算法可以利用请求延迟信息来减轻慢节点的负载，并避开超载下挣扎的主机。
