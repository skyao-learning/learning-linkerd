# 插件

linkerd 构建在模块化插件系统上，以便可以替换各个组件而无需重新编译。这也允许任何人构建定制化插件，实现特定于他们需要的功能。本指南将向您展示如何编写自己的自定义的 linkerd 插件，如何打包它以及如何将它安装在 linker 中。

在本指南中，我们将编写一个自定义 [HTTP 响应分类器][http-response-classifiers]。当然，本指南中的想法也适用于编写自定义identifier，namer，名称解释器，协议或任何其他 linkerd 插件。

本指南中的所有插件代码均可在Github上获得。我建议您在阅读本指南时将代码放在另一个窗口中，以便您可以跟随。

## 概述

我们将编写一个自定义的HTTP响应分类器，它根据一个特殊的响应头而不是HTTP响应代码对响应进行分类。如果header的值为“成功”，我们将把响应视为成功，如果是“重试”，我们将其视为可重试的失败，否则我们将其视为不可重试的失败。

我们将描述如何编写插件，如何构建和打包它，以及如何在linkerd中安装它。

## 编写插件

虽然 linkerd 本身是用 scala 编写的，但是它的插件可以用 scala 或 java 编写。为了演示这个，我们将在 java 中编写我们的插件。为了使我们的插件正常工作，我们将需要3个东西：响应分类器本身，一个配置类和一个配置初始化器。所有插件都遵循这个模式：有实现业务逻辑的类，有配置类，和有配置初始化器。

## 响应分类器

`HeaderClassifier.java` 是响应分类器本身。响应分类器必须继承 `PartialFunction [ReqRep，ResponseClass]`。 每个插件类型都有一个不同的必须实现的接口。例如，namer插件必须实现`Namer`，标识符插件必须实现 `Identifier`。

## 配置类

接下来，我们需要一个类来定义此插件的配置块的结构，并构建响应分类器。我们将调用这个`HeaderClassifierConfig.java`。 请注意，`HeaderClassifierConfig` 必须实现 `ResponseClassifierConfig`。 `ResponseClassifierConfigs` 由Jackson的 linkerd config 的响应分类器部分进行反序列化，其公共成员由相应的json（或yaml）属性填充（在scala中，这将是一个case类）。在我们的例子中，我们有一个名为 `headerName` 的公共成员，它定义了要使用的响应头的名称。

为了满足 `ResponseClassifierConfig` ，我们还必须实现一个名为 `mk()` 的方法，构建响应分类器。

## 配置初始化器

配置初始化器是一个特殊的类，linkerd 在启动时加载，告知 linkerd 关于它可以使用的配置类。我们创建一个名为 [HeaderClassifierInitializer.java][HeaderClassifierInitializer] 的配置初始化器。这个类必须定义一个 `configId` 和一个 `configClass`。 当 linkerd 解析具有 `kind` 属性的配置块时，它将使用该`configId`查找配置初始化器，并尝试将该块反序列化为 `configClass` 的实例。 `HeaderClassifierInitializer` 告诉 linkerd 当它找到一个具有`io.buoyant.headerClassifier` 的 `responseClassifier` 块时，它应该将该块反序列化为 `HeaderClassifierConfig`。

最后，为了使链接器能够在启动时动态加载配置初始化器，我们必须向服务装载器注册它。为此，只需创建一个名为 [META-INF/services/io.buoyant.linkerd.ResponseClassifierConfig][ResponseClassifierInitializer] 的资源文件，并将配置初始化器的完全限定类名添加到该文件。

## 构建和打包

我们使用sbt构建我们的插件和assembly sbt插件，将其打包到一个jar。这是项目的build.sbt文件。注意，我们可以将任何 linkerd 依赖标记为“provided”。 这意味着这些依赖关系将由 linkerd 提供，不需要包含在 plugin jar中。类似地，linkerd 将提供scala标准库，因此我们可以通过设置 `includeScala = false` 来从jar中排除这些。

构建插件jar,运行

```bash
./sbt headerClassifier/assembly
```

## 安装

要在 linkerd 中安装此插件，只需将插件 jar 移动到 linkerd 的插件目录（`$L5D_HOME/plugins`）中即可。 然后在你的 linkerd 配置中的 router 上添加一个 classifier 块：

```bash
routers:
- ...
  responseClassifier:
    kind: io.buoyant.headerClassifier
    headerName: status
```

如果使用 `-log.level=DEBUG` 运行 linkerd，那么应该在启动时看到打印的这行，表示 `HeaderClassifierInitializer` 已经被加载：

```bash
LoadService: loaded instance of class io.buoyant.http.classifiers.HeaderClassifierInitializer for requested service io.buoyant.linkerd.ResponseClassifierInitializer
```

## 试试看

现在 plugins 目录中有我们的插件，我们来试试看。用这个简单的发送所有请求到`localhost：8888` 的配置启动 linkerd。

```bash
routers:
- protocol: http
  dtab: /svc/* => /$/inet/localhost/8888
  responseClassifier:
    kind: io.buoyant.headerClassifier
    headerName: status
  servers:
  - ip: 0.0.0.0
    port: 4140
```

然后，我们将在端口 8888 上启动一个简单的服务器，响应中的状态头指示成功：

```bash
while true; do echo -e "HTTP/1.1 200 OK\r\nstatus: success\r\n" | nc -i 1 -l 8888; done
```

现在我们来发一个请求：

```bash
curl -v localhost:4140
```

通过检查 linkerd 的 metrics，我们可以看到这个请求被归类为成功：

```bash
curl -s localhost:9990/admin/metrics.json?pretty=1 | grep -E 'srv.*(success|failure)'
  "rt/http/srv/0.0.0.0/4140/success" : 1,
```

现在让我们重新启动服务器，并将它修改为设置头为“failure”：

```bash
while true; do echo -e "HTTP/1.1 200 OK\r\nstatus: failure\r\n" | nc -i 1 -l 8888; done
```

并发出另一个请求：

```bash
curl -v localhost:4140
```

现在，当我们检查 linkerd 的 metrics 时，我们会看到这个请求被归类为失败：

```bash
curl -s localhost:9990/admin/metrics.json?pretty=1 | grep -E 'srv.*(success|failure)'
  "rt/http/srv/0.0.0.0/4140/failures" : 1,
  "rt/http/srv/0.0.0.0/4140/failures/com.twitter.finagle.service.ResponseClassificationSyntheticException" : 1,
  "rt/http/srv/0.0.0.0/4140/success" : 1,
```

# 更多信息

如果您对使用或开发 linkerd 插件有任何疑问或想分享您创建的内容，请转到 [linkerd public Slack][slack]中。我们期待与您相逢！

[http-response-classifiers]:  https://linkerd.io/config/1.1.2/linkerd#http-response-classifiers
[HeaderClassifierInitializer]:https://github.com/linkerd/linkerd-examples/blob/master/plugins/header-classifier/src/main/java/io/buoyant/http/classifiers/HeaderClassifierInitializer.java
[ResponseClassifierInitializer]: https://github.com/linkerd/linkerd-examples/blob/master/plugins/header-classifier/src/main/resources/META-INF/services/io.buoyant.linkerd.ResponseClassifierInitializer
[slack]: http://slack.linkerd.io/
