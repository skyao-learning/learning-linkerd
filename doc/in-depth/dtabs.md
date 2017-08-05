# dtabs

Delegation tables/委托表（简称dtabs）是路由规则列表，将 "逻辑路径"（例如流行的冰淇淋店）转换为它所在的“具体名称”（例如，2790 Harrison St， San Francisco，CA 94110）。这是一个我们称之为“解析”的过程，它通过一系列前缀重写发生。

除了本文档，您还可以参考　[Finagle　的　dtab　文档](http://twitter.github.io/finagle/guide/Names.html)。 您还可以通过浏览　http://localhost:9990/delegator　来体验为运行中的　linkerd　实例服务的　dtab　功能。有关 dtab playground 的更多详细信息，请参阅 [“管理”页面](https://linkerd.io/getting-started/admin/#dtab-playground)。

## 路径

最简单的dtab包含一个规则（称为dentry/目录项）:

```bash
/ iceCreamStore => / smitten;
```

这个目录项真的只适用于冰淇淋店，所以规则不适用于路径 `/shoeStore/windowShop/sandals` 。

但是对于路径 `/iceCreamStore/try/allFlavors` ，前缀与目录项的左侧（源）匹配，并用右侧（目标）替换以创建新路径： `/smitten/try/allFlavors`

## 目录项和顺序

Dtabs可以（并且经常做）有多个目录项。例如，我们可以列出几个商店：

```bash
/smitten       => /USA/CA/SF/Octavia/432;
/iceCreamStore => /smitten;
/iceCreamStore => /humphrys;
```

当我们尝试解析一个匹配多个前缀的路径时，优先使用底部的目录项。 所以路径　`/iceCreamStore/try/allFlavors`　将首先解析为 `/humphrys/try/allFlavors`。 然而，如果humphrys 的地址是未知的（在本例中），我们将返回 `/smitten/try/allFlavors`，最终解析为 `/USA/CA/SF/Octavia/432/try/allFlavors`。

## 一步一步的例子

假设有 dtab:

```bash
/iceCreamStore    => /smitten;
/smitten/try      => /smittenLocation/waitInLine/thenTry;
/smittenLocation  => /sanfrancisco/octavia/432;
/california       => /USA/CA;
/sanfrancisco     => /california/SF;
```

和路径:

```bash
/iceCreamStore/try/allFlavors
```

这是解析步骤：

`/iceCreamStore/try/allFlavors` 首先匹配规则 `/iceCreamStore => /smitten;`　并被重写为

`/smitten/try/allFlavors` 继续匹配规则 `smitten/try => /smittenLocation/waitInLine/thenTry;` 并被重写为

`/smittenLocation/waitInLine/thenTry/allFlavors` 继续匹配规则 `/smittenLocation => /sanfrancisco/octavia/432;` 并被重写为

`/sanfrancisco/octavia/432/waitInLine/thenTry/allFlavors` 继续匹配规则 `/sanfrancisco => /california/SF;` 并被重写为

`/california/SF/octavia/432/waitInLine/thenTry/allFlavors` 继续匹配规则 `/california => /USA/CA;` 并被重写为

`/USA/CA/SF/octavia/432/waitInLine/thenTry/allFlavors`!

请注意，每次前缀匹配时，我们从新创建的路径开始，并从底部到顶部再次游历整个dtab。这是有用的，但也容易意外循环！ 考虑以下无限dtab（不用担心，递归调用过多时 Finagle 会退出）：

```bash
/iceCream        => /youScream;
/youScream       => /weAllScream/for;
/weAllScream/for => /iceCream;
```

## 命名器 & 地址

到目前为止，我们只讨论了路径路由。但是为了使 Finagle 成功路由请求，路径必须最终解决为具体的名称。大多数这些具体名称（在Finagle中，它们被称为“绑定地址”）由命名器定义或查找。

Finagle 提供了一个这样的名为 `/$/inet` 的命名器，将两个后续路径段解释为ip地址和端口。所以路径 `/$/inet/127.0.0.1/4140` 将解析到绑定地址 `127.0.0.1:4140`

Linkerd 还为许多不同的服务发现机制提供了一套命名器。 例如 `/#/io.l5d.consul`，`/#/io.l5d.k8s`和 `/#/io.l5d.marathon` 。请在 [linkerd documentation on namers](https://linkerd.io/config/1.1.2/linkerd#namers) 中查看更多关于这些和其他的更多信息。

一旦命名器将路径转换成绑定地址，则路由被认为是完整的，并且未在前缀匹配中使用的任何剩余路径段将保持未使用。作为一个例子，我们定义一个命名器 `/#/routeOnMethod`，它采用下一个路径段，并根据它是一个GET还是POST路由流量。然后对于 `/http/1.1 => /#/routeOnMethod;` 路径 `/http/1.1/GET/host/users` 将被重写到 `/#/routeOnMethod/GET/host/users` ，而前缀 `/#/routeOnMethod/GET` 将解析为绑定地址。其余的段 `/host` 和 `/users` 对请求路由没有任何影响。

然而，命名器并不仅限于解析路径。作为他们最基本的用法，命名器的功能是操作它后面的路径段。考虑将后面两个段相乘并返回数字的命名器 `/#/multiply` 。 对于dtab：

```bash
/byNine  => /#/multiply/9;
/byEight => /#/multiply/8;
/bySeven => /#/multiply/7;
```

The path `/byNine/3` will be rewritten to `/#/multiply/9/3` and finally to `/27`.

## 通配符

当接收到像 `/http/1.1/GET/chocolate/icecream` 这样的路径，在路由请求时我们可能没有兴趣使用每个路径段。 如果所有冰淇淋需要路由到 `/smitten`，它是什么味道没关系。写这个 dtab 的一种方法是列出所有可能的风格：

```bash
/http/1.1/GET/chocolate/icecream => /smitten;
/http/1.1/GET/vanilla/icecream => /smitten;
/http/1.1/GET/rockyroad/icecream => /smitten;
/http/1.1/GET/strawberry/icecream => /smitten;
/http/1.1/GET/mintchip/icecream => /smitten;
```

更简单和更优雅的解决方案是使用通配符替换风味段，该通配符将匹配该段的任何字符串。

```bash
/http/1.1/GET/*/icecream => /smitten;
```

## 备选，并列和权重

当两个目录项具有相同的前缀时，我们将它们称为备选。 我们早先看到一个例子，再看一次：

```bash
/smitten       => /USA/CA/SF/Octavia/432;
/iceCreamStore => /smitten;
/iceCreamStore => /humphrys;
```

备选也可以使用管道操作符指定：

```bash
/smitten       => /USA/CA/SF/Harrison/2790;
/iceCreamStore => /humphrys | /smitten;
```

在这两个例子中，humphrys 是我们尝试解决的第一个冰淇淋店。但是如果没有找到地址，我们继续处理smitten（如果smitten也没有找到，则整个路由操作失败 - 没有人得到冰淇淋）。您可以指定任意数量的备选　`/humphrys | /smitten | /birite | /three-twins`...

Dtabs 还支持并列,使用以下语法的 `/iceCreamStore => /humphrys & /smitten`。 在这个例子中，我们有平等的机会将路径路由到任一个商店。如果我们希望比另一个更有可能进入一个商店，我们可以为每个路径添加权重：

```bash
/smitten       => 3 * /SF/Octavia/432 & 1 * /SF/California/2404;
/iceCreamStore => 0.7 * /humphrys & 0.3 * /smitten;
```

权重可以是小数或整数。

## 负面，失败和空解析

如果命名器不能找到具体的地址，它会返回一个负面的解析。 这是向 Finagle 发出信号，这个路径无效，如果有任何替代路径可以尝试，现在是合适的时间。如果所有路径都为负面，Finagle 会抛出错误。这种回退逻辑可以用符号进行测试,”～“ 是 Finagle 理解为负面解析的符号。例如：

```bash
/iceCreamStore => ~ | /smitten;
```

如果我们要在检查任何备选路径之前停止，我们应该使用失败而不是负面。失败指定使用　`/$/fail`　甚至更短 `！`，像在这个　dtab　中，我们路由到　smitten　或失败：

```bash
/iceCreamStore => /smitten | !;
```

命名器有时也会返回失败的解析。例如对于路径 `/multiply/cats/dogs` 命名器 `/multiply` 会返回失败。

最后还有一种称为空的解析。它通过　`/$/nil` 或者 `$` 调用，通常仅在测试场景中使用。

