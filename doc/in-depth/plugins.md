# 插件

linkerd 构建在模块化插件系统上，以便可以替换各个组件而无需重新编译。这也允许任何人构建定制化插件，实现特定于他们需要的功能。本指南将向您展示如何编写自己的自定义的 linkerd 插件，如何打包它以及如何将它安装在 linker 中。

在本指南中，我们将编写一个自定义 [HTTP 响应分类器](https://linkerd.io/config/1.1.2/linkerd#http-response-classifiers)。当然，本指南中的想法也适用于编写自定义identifier，namer，名称解释器，协议或任何其他 linkerd 插件。

本指南中的所有插件代码均可在Github上获得。我建议您在阅读本指南时将代码放在另一个窗口中，以便您可以跟随。

