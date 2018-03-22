---
uid: addressing
title: Actor References, Paths and Addresses
---

# Actor 引用, 路径和地址

本章描述了actors是如何在一个可能分布的参与者系统中被识别和定位的. 这关系到中心思想 [Actor Systems](xref:actor-systems) 形成内在的监督层次和在跨多个网络节点的位置上，actors之间的通信是透明的

![Actor path](/images/ActorPath.png)

上面的图片显示了一个actor系统中最重要的实体之间的关系，请详细阅读。.

## 什么是 Actor Reference?
一个 actor 引用是 `ActorRef`的一个子类型, 最主要的用途是支持发送消息到它代表的actor。每一个 actor 通过`Self`属性访问它的规范 (本地l)引用，这个引用也是缺省包含在它发到其他actor的所有消息中，作为sender . 反过来，在消息处理过程中， actor通过Sender方法访问当前收到的消息的发送者的引用。 

有几种不同类型的actor引用，这取决于参与者系统的配置：

- 单纯本地actor引用被actor 系统使用，没有配置支持网络功能 如果通过网络连接发送到远程CLR，这些Actor引用将不起作用.
- 启用远程处理时的本地Actor引用由Actor系统使用，该Actor支持表示同一CLR内的Actors的那些引用的联网功能. 为了在发送到其他网络节点时也可以访问，这些引用包括协议和远程寻址信息.
- 存在本地Actor引用的子类型作为路由器使用它的逻辑结构与上述的本地引用相同，但是发送消息直接发送给他们的一个孩子。
-远程Actor引用代表使用远程通信可访问的Actor，即向他们发送消息将透明地序列化消息并将消息发送给远程CLR.
- 出于各种实际目的，有几种特殊类型的actor 引用，行为类似于本地Actor引用：
  -`PromiseActorRef`是为了通过某个actor的回应完成的`Task`的特殊表示。`ICanTell.Ask`创建这个actor引用。
  -`DeadLetterActorRef`是Akka死信函服务的默认实现，Akka将目的地被关闭或不存在的所有的消息路由到这个地址。
  - ``EmptyLocalActorRef`是Akka在查找不存在的本地actor路径时返回的内容：它相当于一个`DeadLetterActorRef`，但它保留它的路径，以便Akka可以通过网络发送它并将它与其他现有的actor引用进行比较 那条道路，其中一些可能是在actor死亡之前获得的。
- 然后有一些你永远不会看到的一次性内部实现：
  - 有一个actor的引用不代表actor，但只作为根监护人的伪监督者，我们称之为“走过时空泡沫的人the one who walks the bubbles of space-time”。
  - 在实际启动actor创建设施之前启动的第一个日志记录服务是假的actor引用，它接受日志事件并将它们直接打印到标准输出; 它是`Logging.StandardOutLogger`。

## 什么是 Actor Path?
由于actor是以严格等级的方式创建的，因此存在一系列独特的actor名字 ，这些actor名字通过递归地跟随儿童和父母之间的监督链接向下Actor系统的根部而给出的。这个序列可以看作是将文件夹包含在一个文件系统中，因此我们采用名称“path”来引用它，尽管actor层次结构与文件系统层次结构有一些根本的区别。

一个actor路径由一个锚点组成，该锚点标识actor系统，然后是从根监护人到指定actor的路径元素的连接; 路径元素是遍历actor的名称，并用斜杠分隔。

### Actor引用和路径的区别?


 actor引用指定单个actor，引用的生命周期与actor的生命周期匹配; 一个actor路径代表一个actor可能会或可能不会居住的名字，而路径本身没有一个生命周期，它永远不会失效。您可以创建actor路径而不创建actor，但不能创建演员引用而不创建相应的Actor。您可以创建一个actor，终止它，然后使用相同的actor路径创建一个新actor。新创建的actor是actor的新角色。这不是同一个actor。一个actor引用旧的化身不适用于新的化身。发送给旧actor引用的消息即使具有相同的路径，也不会传递给新的actor。Messages sent to the old actor reference will not be delivered to the new incarnation even though they have the same path.

### actor路径锚 

每个actor路径都有一个地址组件，描述了相应actor可访问的协议和位置，后面跟着层次结构中actor的名称. 例子如下:

````csharp
"akka://my-sys/user/service-a/worker1"                   // purely local
"akka.tcp://my-sys@host.example.com:5678/user/service-b" // remote
````
这儿, `akka.tcp` 是默认的远程传输; 其他传输是可插拔的。使用UDP的远程主机可以通过使用akka.udp进行访问. 主机和端口部分的解释（即例子中的``serv.example.com：5678``）取决于使用的传输机制，但它必须遵守URI结构规则。

### 逻辑 Actor 路径
通过跟随家长监督链接获得的独特路径称为逻辑行动者路径。该路径完全匹配actor的创建祖先，所以只要actor系统的远程配置（并且使用该路径的地址组件）被设置，它就是完全确定的。

### 物理 Actor 路径
虽然逻辑actor路径描述了一个actor系统内的功能位置，但基于配置的远程部署意味着actor可以在不同于其父节点的不同网络主机上创建，即在不同的actor系统内。

 在这种情况下，跟随来自根监护人的actor路径需要穿越网络，这是一种代价高昂的操作。因此，每个actor都有一个物理路径，从实际actor对象所在actor系统的根监护人开始。在查询其他actor时使用此路径作为发件人引用将让他们直接回复此actor，从而最大限度地减少由路由引发的延迟。一个重要的方面是物理actor路径从不跨越多个actor系统或CLR。这意味着如果一位actor的祖先被远程监督，则actor的逻辑路径（监督层级）和物理路径（actor部署）可能会发生分歧。

### Actor路径别名（ path alias） 或者符号链接（symbolic link）?
However, you should note that actor hierarchy is different from file system hierarchy. You cannot freely create actor paths like symbolic links to refer to arbitrary actors. 正如在一些真实的文件系统中，您可能会想到actor的“路径别名”或“符号链接”，即一个actor可能使用多条路径到达。但是，您应该注意，actor层次结构与文件系统层次结构不同。你不能自由地创建像符号链接那样的actor路径来引用任意actor。如上述逻辑和物理actor路径部分所述，actor路径必须是表示监督层次结构的逻辑路径，或者表示actor部署的物理路径。

## 如何获得 Actor 引用?
关于如何获得actor引用有两个一般类别：创建actor或者查看actor，后者的功能来自于从具体actor路径创建actor引用和查询逻辑actor层次结构创建actor引用的两种风格。.

###创建Actors

Actor系统通常通过使用“ActorSystem.ActorOf”方法在监护人actor下创建actor，然后在创建的actor中使用“ActorContext.ActorOf”来产生actor树来启动。这些方法返回对新创建的actor的引用。每个actor都可以直接访问（通过它的`ActorContext`）它的父代应用，它自己和它的子代。这些引用可以在消息内发送给其他角色，使他们能够直接回复。

###通过具体路径查找Actor

此外，actor引用可以使用`ActorSystem.ActorSelection`方法进行查找。该选择可以用于与所述actor进行通信，并且在传递每个消息时查找与该选择对应的actor。要获取绑定到特定actor的生命周期的ActorRef，您需要向actor发送消息（例如内置的“Identify”消息），并使用actor回复消息的“Sender”。

###绝对与相对路径 

除了ActorSystem.actorSelection之外，还有`ActorContext.ActorSelection`，它可以在任何actor中作为`Context.ActorSelection`使用。这产生了一个actor选择，就像它在“ActorSystem”上的双胞胎一样，但是不是从当前actor树根开始查找路径,而是从当前actor 开始。可以使用由两个点（“..”）组成的“路径”元素来访问父actor。例如，您可以将消息发送给特定的兄弟姐妹：

````csharp
Context.ActorSelection("../brother").Tell(msg);
````
绝对路径当然也可以通常的方式在上下文中查找，即
````csharp
Context.ActorSelection("/user/serviceA").Tell(msg);
````
将按预期工作。

 ###查询逻辑Actor层次结构 
 由于actor系统形成层次结构的文件系统，路径匹配可能与Unix shell支持的方式相同：您可以用通配符（«*»和«？»）替换（部分路径元素名称）来制定 一个可以匹配零个或多个实际actor的选择。因为结果不是单个actor引用，所以它具有不同的类型'ActorSelection`，并且不支持`IActorRef`完成的全部操作。可以使用“ActorSystem.ActorSelection”和“IActorContext.ActorSelection”方法来制定选择，并支持发送消息：

```csharp
Context.ActorSelection("../*").Tell(msg);
```
会发送消息给所有兄弟姐妹，包括当前的actor。

##总结：`ActorOf`与`ActorSelection` > 

[！NOTE] >以上部分详细描述的内容可以简单归纳和记忆如下： > *“ActorOf”只创建一个新的actor，并将其创建为调用此方法的上下文的直接子对象（可以是任何actor或actor系统）。
> *“ActorSelection”只有在传递消息时才会查找现有actor，即不会创建actor，或者在创建选择时验证actor是否存在。



##Actor 引用和路径相等 
“ActorRef”的对等符合“ActorRef”对应于目标actor化身的意图。当两个actors参照具有相同的路径并且指向相同的actor化身时，它们被比较是相等的。指向已终止的actor的引用不会等于指向具有相同路径的另一个（重新创建的）actor的引用。请注意，由故障引发的actor重启仍然意味着它是同一个 actor的化身，即对'ActorRef`的使用者来说重启是不可见的。如果您需要跟踪集合中的actor引用，并且不关心具体actor角色，则可以使用“ActorPath”作为关键字，因为在比较actor路径时不会考虑目标actor的标识符。

 ##重新使用actor路径
当actor被终止时，其引用将指向死信信箱，“DeathWatch”将发布其最终过渡，并且通常不会再次复生（因为actor生命周期不允许这样做）。

尽管可以在以后使用相同的路径创建actor，但仅仅是因为如果不保留所有actor创建的集合都可用，就不可能执行相反的操作，这不是一种好的做法：使用ActorSelection发送的消息对于“死亡”的actor突然重新开始工作，但没有任何保证在这个过渡和任何其他事件之间进行排序，因此新路径的居民可能会收到发往前一位房客的信息。

在非常特殊的情况下做这件事可能是对的，但是确保将这个处理恰好限制在actor的监督者身上，因为那是唯一可以可靠地检测到正确注销名字的actor，在此之前创建新的孩子会失败。

在测试期间，当测试对象依赖于在特定路径上被实例化时，也可能需要它。

在这种情况下，最好是模拟它的主管，以便将终止消息转发到测试过程中的适当位置，使后者能够等待正确注销该名称。

 ##与远程部署的相互作用
 
 当一个actor创建一个孩子时，actor系统的部署者将决定新的actor是否驻留在同一个CLR中或另一个节点上。在第二种情况下，actor的创建将通过网络连接触发，以在不同的CLR中发生，并因此在不同的actor系统中发生。远程系统会将新actor放置在为此目的而保留的特殊路径下，并且新actor的监督者将成为远程actor引用（表示触发其创建的actor）。在这种情况下，context.parent（监督引用）和context.path.parent（actor路径中的父节点）不代表同一个actor。但是，在主管中查找孩子的名字将在远程节点上找到它，保持逻辑结构，例如， 发送给未解决的actor引用时。

![Remote Deployment](/images/RemoteDeployment.png)


##地址部分作用？ 
当通过网络发送Actor引用时，它由其路径表示。因此，路径必须完整地编码将消息发送给潜在的actor所需的所有信息。这是通过在路径字符串的地址部分编码协议，主机和端口来实现的。当一个actor系统从远程节点接收到一个actor路径时，它会检查该路径的地址是否与该actor系统的地址相匹配，在这种情况下，它将被解析为actor的本地引用。否则，它将由远程actor 引用表示。

 ##Actor路径的顶级范围
在路径层次结构的根部驻留着根监护人，在其上可找到所有其他actor;它的名字是``/“`。

下一个级别包括以下内容：

*`“/ user”`是所有用户创建的顶级actor的监护人actor;在这个下面找到使用`ActorSystem.ActorOf`创建的actor。

*`“/ system”`是所有系统创建的顶级actor的监护人actor，例如，日志监听器或通过配置在actor系统启动自动部署的actors。

*`“/ deadLetters”`是死信actor，这是所有发送给已停止或不存在actor的消息被重新路由的地方（尽力而为：即使在本地CLR中也可能丢失消息）。

*`“/ temp”`是所有短命的系统创建actor的守护者，例如那些在ActorRef.ask的实现中使用的。

*``/ remote“`是一个虚拟路径，在这个虚拟路径下，所有驻留actors监督者是远程actor引用
为这样的actor构建名字空间的需求来自一个中心且非常简单的设计目标：层次结构中的所有内容都是actor，所有actors的功能都是相同的。

因此，您不仅可以查看您创建的actors，还可以查看系统监护人并向其发送一条消息（在此情况下它将尽职地丢弃）。

这个强大的原理意味着没有怪癖记忆，它使整个系统更加统一和一致。

如果您想了解更多关于actor系统的顶层结构，请查看 [The Top-Level Supervisors](xref:supervision#the-top-level-supervisors)。
