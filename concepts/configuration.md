---
uid: configuration
title: Configuration
---

# Akka.NET 配置

*Quoted from [Akka.NET Bootcamp: Unit 2, Lesson 1 - "Using HOCON Configuration to Configure Akka.NET"](https://github.com/petabridge/akka-bootcamp/tree/master/src/Unit-2/lesson1 "Using HOCON Configuration to Configure Akka.NET")*


Akka.NETt利用 HOCON配置格式，允许您配置任何级别粒度的tAkka.NET应用。

#### 什么是 HOCON?
HOCON (Human-Optimized Config Object Notation) 是一种可扩展的灵活的配置格式
这将允许您配置一切包括 Akka.NET's IActorRefProvider 实现 ，日志，网络传输，更普遍的个体actors如何部署。.

 HOCON  返回的值是强类型的（即，你可以返回`int`，`Timespan`等.

#### HOCON能做什么？
HOCON允许您在App.config和Web.config中在很难读取XML嵌入轻松易读的配置。也可以让你通过配置senction路径查询配置，并且这些配置section暴露的强类型和解析值可以用在你的应用程序中。

HOCON 也可以让你的内嵌和/或链接配置sections，创建层的粒度和为您提供一个语义命名空间配置.

#### HOCON 通常用来做什么?
HOCON通常用来调试日志设置，使能特殊模块 (如 `Akka.Remote`), 或者配置实施，如 为特定actor使用的`Dispatcher` or `Router`.

例如，让我们使用HOCON 配置一套 `ActorSystem` :

```csharp
var config = ConfigurationFactory.ParseString(@"
akka.remote.dot-netty.tcp {
    transport-class = ""Akka.Remote.Transport.DotNetty.DotNettyTransport, Akka.Remote""
    transport-protocol = tcp
    port = 8091
    hostname = ""127.0.0.1""
}");

var system = ActorSystem.Create("MyActorSystem", config);
```

从例子中可以看出,一个 HOCON `Config` 对象能够从 一个 `string`进行解析，解析使用 `ConfigurationFactory.ParseString`方法. 当你有一个 `Config` 对象，在`ActorSystem.Create` 方法中传递给你的 `ActorSystem`.

#### 部署?那是什么t?
Deployment 是一个模糊的概念，但它与 HOCON紧密联系. 一个 actor部署，亦即actor的实例化并在 `ActorSystem`的某个位置 投入服务.

当一个actor 在 `ActorSystem`中实例化，它能够部署在两个位置的一个：在本地进程，还是在另一个进程（这就是`Akka.Remote` 的功能）

当一个actor 在 `ActorSystem`中部署, 它有一系列的配置项. 这些配置控制actor大批的行为选型，比如：这个actor是否是一个router?它将使用什么 `Dispatcher`?它将拥有何种类型的 mailbox？

我们这儿不对所有的选项的含义进行阐述，但是现在知道关键的一点是 `ActorSystem`部署一个 actor并提供服务的配置均可以在 HOCON中设置. *

***这也意味你可以戏剧性地改变actor的行为（通过改变这些设置），而不必实际触及actor代码本身。.***

灵活配置型!

#### HOCON can be used inside `App.config` and `Web.config`
从`string·解析HOCON`方便小配置的部分，但是如果你希望能够利用 [Configuration Transforms for `App.config` and `Web.config`](https://msdn.microsoft.com/en-us/library/dd465326.aspx) 和其他一些`System.Configuration` 名字空间中好的工具  ?

结果是，你能够在这些配置文件中使用 HOCON 

以下是在 `App.config`中使用HOCON的例子:

```xml
<?xml version="1.0" encoding="utf-8" ?>
<configuration>
  <configSections>
    <section name="akka" type="Akka.Configuration.Hocon.AkkaConfigurationSection, Akka" />
  </configSections>

  <akka>
    <hocon>
      <![CDATA[
          akka {
            # here we are configuring log levels
            log-config-on-start = off
            stdout-loglevel = INFO
            loglevel = ERROR
            # this config section will be referenced as akka.actor
            actor {
              provider = remote
              debug {
                  receive = on
                  autoreceive = on
                  lifecycle = on
                  event-stream = on
                  unhandled = on
              }
            }
            # here we're configuring the Akka.Remote module
            remote {
              dot-netty.tcp {
                  transport-class = "Akka.Remote.Transport.DotNetty.DotNettyTransport, Akka.Remote"
                  #applied-adapters = []
                  transport-protocol = tcp
                  port = 8091
                  hostname = "127.0.0.1"
              }
            log-remote-lifecycle-events = INFO
          }
      ]]>
    </hocon>
  </akka>
</configuration>
```

然后，我们使用下面代码加载配置到我们的`ActorSystem`:

```csharp
var system = ActorSystem.Create("MySystem"); //automatically loads App/Web.config
```

#### HOCON 配置支持回退 
这是一个 `Config` 类强大的特性，可以很便利并在很多的产品中有运用。

HOCON 支持"fallback" 配置 - 这很容易在字面上解释.

![Normal HOCON Config Behavior](/images/hocon-config-normally.gif)

为了建立上图中的情形，我们创建了含三个 回退的`Config` 对象，语法如下:

```csharp
var f0 = ConfigurationFactory.ParseString("a = bar");
var f1 = ConfigurationFactory.ParseString("b = biz");
var f2 = ConfigurationFactory.ParseString("c = baz");
var f3 = ConfigurationFactory.ParseString("a = foo");

var yourConfig = f0.WithFallback(f1)
				   .WithFallback(f2)
				   .WithFallback(f3);
```

如果你向 HOCON 对象请求键 "a"的值，使用以下的代码片段:

```csharp
var a = yourConfig.GetString("a");
```

然后内部 HOCON引擎将第一个包含key `a`定义的 HOCON文件. 这种情况下，值为 `f0`的 "bar".

####  为什么 "a"返回的不是 "foo"?
原因是 HOCON 只通过fallbak `Config' 对象检索在`Config`链中是否没有发现更早的匹配. 如果顶层 `Config`对象有一个 `a`的匹配,   fallbacks就不被检索. 在这种情况下, 在 `f0`中发现了`a`的一个匹配， 因此`f3`中的 `a=foo` 不会被访问.

####当缺少 HOCON key会如何?
如果没有在`f0`和`f1`中定义`c`，运行下面的代码会发生什么情况?

```csharp
var c = yourConfig.GetString("c");
```

![Fallback HOCON Config Behavior](/images/hocon-config-fallbacks.gif)

在这种情况下 `yourConfig` 将 fallback 2次到 `f2`并返回"baz" 作为 key `c`的值.
