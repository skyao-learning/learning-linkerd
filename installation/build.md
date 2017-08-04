# 构建项目

## 安装scala

http://www.scala-lang.org/download/

简单起见，我们选择在idea中使用scala，参考官方文档：

- [GETTING STARTED WITH SCALA IN INTELLIJ](http://www.scala-lang.org/documentation/getting-started-intellij-track/getting-started-with-scala-in-intellij.html)

先在 idea 中安装 scala 插件。

## 设置sbt的ivy私库

sbt下载依赖的速度非常的慢，为了加速，需要设置sbt的ivy私库为本地私库。

修改/添加文件 `~/.sbt/repositories`:

```bash
[repositories]
  local
  dataman:http://192.168.31.21:8081/nexus/content/groups/public/
  dataman-ivy:http://192.168.31.21:8081/nexus/content/groups/public/, [organization]/[module]/(scala_[scalaVersion]/)(sbt_[sbtVersion]/)[revision]/[type]s/[artifact](-[classifier]).[ext]
```

## 设置代理

linkerd 项目需要从下面的地址中获取部分依赖：

https://maven.twttr.com/

但是这个地址时被强的，无法访问，因此解析依赖时会失败，报错"connection reset"。

~~为此需要设置 idea 的代理~~

备注： idea 的代理对sbt无效。

修改

```bash
export SBT_OPTS="$SBT_OPTS -Dhttp.proxyHost=127.0.0.1 -Dhttp.proxyPort=8123"
```

## 导入项目

git clone 下来 linkerd 源码，然后在 idea 中导入 linkerd 的项目。

备注：未能build成功，总是报错说找不到依赖。放弃。。。
