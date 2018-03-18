---
uid: supervision
title: Supervision
---

#监督

本文概述了监督理念和对于运行中的时akka.net acto背后意味着什么。

##监督意味着什么

正如Actor系统描述中所描述的，Actor之间有一种依赖关系：监督者将任务委托给从属Actor，因此必须对他们的失败作出反应。当从属Actor检测到故障（即抛出异常）时，它会暂停自身及其所有下属，并向其监督者发送消息，发出信号失败。根据所监督的工作的性质和失败的性质，主管可以选择以下四种方案：

- **恢复Resume**从属Actor，保持其积累的内部状态。
- **重新启动Restart**从属Actor，清除其积累的内部状态。
- **永久停止Stop**从属Actor
- **升级失败Escalate**到层次结构中的下一个父节点，从而失败自身


重要的是，一般认为Actor作为一个监督层次的一部分，它解释了第四种选择的存在（作为一个监督者也隶属另一个上级监督者）并且在第一三启示：恢复Actor并恢复其所有下属Actor，Actor需要重新启动其所有下属（见下文更多的细节），同样终止Actor也将终止其所有下属。应该指出的是，该untypedactor prerestart钩的默认行为是重新启动前终止所有的下属，但这个钩子可以重写；递归重启应用后钩已执行的所有子Actors。

每个监督者都配置了一个函数，将所有可能的失败原因（即异常）转换成上面给出的四个选项中的一个；特别是，这个函数不把失败的参与者的身份作为输入。很容易提出一些结构上的例子，而这些结构似乎不够灵活，例如希望将不同的策略应用于不同的下属。在这一点上，必须理解，监督是关于形成递归故障处理结构的。如果你试图在一个层面上做得太多，这就很难解释，因此在这种情况下推荐的方法是增加一个监督级别。

akka.net实现特定形式称为“家长监督”。Actors只能由其他Actpr创建，其中顶级Actor由库提供，而每一个创建的Actor由其父代监督。这种限制使得Actor监督层级的形成是隐含的，并鼓励健全的设计决策。应该指出的是，这也保证了Actor不能孤立或连接到外部监督者，否则可能捕获他们。此外，这将产生一个自然的和干净的关机过程（子树）Actor应用程序。

> [！警告]
>与监督有关的Parent-Child通信发生于具有与用户消息分离的独立邮箱的特殊系统消息中。这意味着监管的相关事件相对于普通邮件的不确定性要求。一般来说，用户不能影响正常消息和故障通知的顺序。细节和例子见[讨论：消息次序小姐]（(xref:message-delivery-reliability#discussion-message-ordering)）。

##顶层监督者

![Top level supervisors](/images/TopLevelSupervisors.png)

Actor系统在创建作过程中至少会产生三个actor，如上图所示。For more information about the consequences for actor paths see [Top-Level Scopes for Actor Paths](xref:addressing#top-level-scopes-for-actor-paths).

### `/user`: The Guardian Actor

最可能与之交互的Actor是所有用户创建的Actor的父Actor，监护人称为“用户”。使用`system.ActorOf()`创建Actor是这个Actor的Actor。这意味着，当守护终止时，系统中所有正常的Actors也将被关闭。这也意味着，监护人的监督策略决定了如何监督顶级正常行为人。可以配置使用的设置`akka.actor.guardian-supervisor-strategy`主管战略，以完全限定类名一` SupervisorStrategyConfigurator `。当守护报告一个失败，根守护的反应会终止的守护者，这实际上会关闭整个Actor系统。

### `/system`: The System Guardian

为了实现一个有序的关闭序列，在所有正常行为终止时，日志记录仍然保持活动，即使日志记录本身是使用Actor实现的，这个特殊的守护者也被引入了。这是通过让系统守护者监视用户监护人并在接收到“终止”消息后启动自己的关机来实现的。顶级Actor监督系统使用的策略将无限期地对所有类型的异常，除了` actorinitializationexception `和` actorkilledexception `重启，这将解除子Actor的问题。所有其他异常都会升级，这将关闭整个Actor系统。

### `/`: The Root Guardian

根的守护者是所有所谓的“顶层”Actor的大家长和监督所有特殊的Actor在 [Top-Level Scopes for Actor Paths](xref:addressing#top-level-scopes-for-actor-paths)使用` supervisorstrategy。stoppingstrategy `，其目的是在任何类型的异常终止的子Actor。所有其他throwables将升级…但谁呢？因为每一个真正的Actor都有一个监督者，所以根本监护人的主管不能成为真正的Actor。因为这意味着它是“在泡沫之外”，它被称为“泡沫行者”。这是一种人工合成的` IAtorRef `在遇到麻烦的第一个迹象，一旦守护者完全终止，Actor系统的终止状态设置为真。（所有Actor递归停止）。

## What Restarting Means

当一个Acror在处理某条消息时失败时，失败的原因分为三类：

* 接收特定消息的系统错误（即编程）错误
* （临时）在处理消息期间使用的外部资源失败
* Actor内部崩溃状态

除非失败是特别可识别的，否则第三个原因不能排除，这就导致了内部状态需要被清除的结论。如果监督者决定，其他的子Actor或本身不受不受腐败影响，比如由于错误的内核模式，因此最好重启孩子Actor自觉应用。这是通过创建的基本` UntypedActor `类和取代失败的实例与新鲜的一个子Actor的` IActorRef `新实例进行；这样的能力是一个Actor在特殊封装引用的原因。然后，新Actor重新处理其邮箱，这意味着重启在Actor本身之外是不可见的，明显的例外是失败发生的消息没有被重新处理。

重新启动期间事件的精确顺序如下：

1. 暂停Actor（这意味着它将不处理正常的消息，直到恢复），并递归地挂起所有的子Actor。
2. 调用旧实例的` PreRestart `钩（默认发送终止请求所有的子Actor 调用PostStop）
3. 等所有的子Actor被要求终止（使用`Context.Stop() `）在` PreRestart `实际上终止；这就像所有的Actor都是非阻塞的操作，从最后一个被杀害的Actor 终止通知将对下一步进展的影响。
4. 通过调用原始提供的工厂再次创建新的Actor实例。
5. 调用` PostRestart `在新实例（默认情况下还要求` PreStart `）
6. 将重新启动请求发送给在步骤3中未被杀死的所有Actor ；重新启动的Actor 将按照步骤2递归地执行相同的过程。
7. 恢复Actor。

## What Lifecycle Monitoring Means 生命周期监控
> [!NOTE]
> Lifecycle Monitoring in Akka.NET is usually referred to as `DeathWatch`


与上面描述的Parent-Child之间的特殊关系相反，每个Actor都可以监视其他Actor。由于Actor从创建到重新启动，在受影响的监督者之外是看不见的，唯一可以监视的状态变化是从生到死的转变。因此，监视被用来将一个Actor与另一个Actor绑在一起，以便它可以对另一个Actor的终止作出反应，而不是对失败作出反应的监督。

生命周期监控使用`Terminated`被Actor 收到消息来实现的，其中的默认行为是抛出一个特殊的` DeathPactException `如果不处理。为了开始听的终止信息，调用` ActorContext.Watch(targetActorRef)`。停止听，调用`ActorContext.Unwatch(targetActorRef)`。一个重要的特性是，不管监视请求和目标终止发生的顺序如何，消息将被传递，也就是说，即使在注册时目标仍然是死的，仍然可以得到消息。

如果监督者不能简单地重新启动其子节点并终止它们，例如监视程序初始化期间出现错误，则监视尤其有用。在这种情况下，它应该监控这些Actors，并重新创建他们或安排自己在以后的时间重试。

另一个常见的用例是，在缺少外部资源的情况下，一个Actor需要失败，外部资源也可能是它自己的孩子之一。如果第三方使用`system.Stop(child)`方法或发送` poisonpill `终止孩子的，监督者可能也会受影响。

###延迟重启与backoffsupervisor模式

提供一个内置的模式`Akka.Pattern.BackoffSupervisor`实行所谓的指数退避的监管策略，启动子Actor，当它再次失败，每次都有一个成长的时间之间的延迟启动。

当启动的角色失败时，此模式非常有用，因为一些外部资源不可用，我们需要给它一些时间重新启动。一个主要的例子时，这是有用的当``UntypedPersistentActor `失败（停止）与持续性失败，这表明数据库可能会下降或超载，在这种情况下，它是有意义的给它一点时间恢复启动前残存的Actor。


下面的C #片段显示了如何创建一个退避监督将开始给予EchoActor后已经停止因为失败，增加的时间间隔3, 6, 12，24和最后30秒：

```csharp
var childProps = Props.Create<EchoActor>();

var supervisor = BackoffSupervisor.Props(
    Backoff.OnStop(
        childProps,
        childName: "myEcho",
        minBackoff: TimeSpan.FromSeconds(3),
        maxBackoff: TimeSpan.FromSeconds(30),
        randomFactor: 0.2));

system.ActorOf(supervisor, "echoSupervisor");
```

U使用`randomFactor`向退避间隔添加一点额外的方差是强烈建议，以避免在时间相同的点多的演员重新开始，例如因为他们停止由于共享资源，如数据库去，重新启动后同样配置的间隔。通过在重新启动间隔中添加额外的随机性，参与者将开始稍微不同的时间点，从而避免大量的流量冲击恢复的共享数据库或其他他们需要接触的资源。

的`Akka.Pattern.BackoffSuperviso `Actor也可以配置为延迟当Actor崩溃和监管策略决定了它应该重启重启后的演员。

下面的C #片段显示了如何创建一个退避主管将开始给予回声演员后也不敢因为一些例外，在增加的时间间隔3, 6, 12，24和最后30秒：
```csharp
var childProps = Props.Create<EchoActor>();

var supervisor = BackoffSupervisor.Props(
    Backoff.OnFailure(
        childProps,
        childName: "myEcho",
        minBackoff: TimeSpan.FromSeconds(3),
        maxBackoff: TimeSpan.FromSeconds(30),
        randomFactor: 0.2));

system.ActorOf(supervisor, "echoSupervisor");
```

的`Akka.Pattern.BackoffOptions`可以用来自定义后退主管Actor的行为，下面是一些例子：

```csharp
var supervisor = BackoffSupervisor.Props(
    Backoff.OnStop(
        childProps,
        childName: "myEcho",
        minBackoff: TimeSpan.FromSeconds(3),
        maxBackoff: TimeSpan.FromSeconds(30),
        randomFactor: 0.2)
        .WithManualReset() // the child must send BackoffSupervisor.Reset to its parent
        .WithDefaultStoppingStrategy()); // Stop at any Exception thrown
```
上面的代码中设置一个后退的主管需要儿童演员送`Akka.Pattern.BackoffSupervisor.Reset`消息给它的父当消息处理成功，重新回到了。它还使用默认的停止策略，任何异常都会导致孩子停止。


```csharp
var supervisor = BackoffSupervisor.Props(
    Backoff.OnStop(
        childProps,
        childName: "myEcho",
        minBackoff: TimeSpan.FromSeconds(3),
        maxBackoff: TimeSpan.FromSeconds(30),
        randomFactor: 0.2)
        .WithAutoReset(TimeSpan.FromSeconds(10))
        .WithSupervisorStrategy(new OneForOneStrategy(exception =>
        {
            if (exception is MyException)
                return Directive.Restart;
            return Directive.Escalate;
        })));
```
上面的代码中设置一个后退主管重启孩子后后退，如果` myexception `抛出任何异常，将升级。如果孩子在10秒内没有抛出任何错误，则自动复位。


## One-For-One Strategy vs. All-For-One Strategy
There are two classes of supervision strategies which come with Akka: `OneForOneStrategy` and `AllForOneStrategy`. Both are configured with a mapping from exception type to supervision directive and limits on how often a child is allowed to fail before terminating it. The difference between them is that the former applies the obtained directive only to the failed child, whereas the latter applies it to all siblings as well. Normally, you should use the `OneForOneStrategy`, which also is the default if none is specified explicitly.

有两类监管策略，跟阿亮：` oneforonestrategy `和` allforonestrategy `。两者都配置了从异常类型到监视指令的映射，并限制了在终止之前允许孩子失败的频率。它们之间的区别在于前者只向失败的孩子应用所获得的指令，而后者则将其应用于所有的兄弟姐妹。通常情况下，你应该使用` oneforonestrategy `，如果没有明确指定，这也是默认的。




![One for one](/images/OneForOne.png)

The `AllForOneStrategy` is applicable in cases where the ensemble of children has such tight dependencies among them, that a failure of one child affects the function of the others, i.e. they are inextricably linked. Since a restart does not clear out the mailbox, it often is best to terminate the children upon failure and re-create them explicitly from the supervisor (by watching the children's lifecycle); otherwise you have to make sure that it is no problem for any of the actors to receive a message which was queued before the restart but processed afterwards.
的` allforonestrategy `如果儿童乐团如此紧密的它们之间的依赖关系是适用的，那一个失败的一个孩子会影响其他的功能，即它们之间有着千丝万缕的联系。由于启动不清理邮箱，这往往是最好的解除孩子在失败和重新明确由主管创建（通过看孩子们的生命周期）；否则，你必须确保它是没有问题的任何演员接收信息，但排队才重启处理后。



![All for one](/images/AllForOne.png)

Normally stopping a child (i.e. not in response to a failure) will not automatically terminate the other children in an all-for-one strategy; this can easily be done by watching their lifecycle: if the `Terminated` message is not handled by the supervisor, it will throw a `DeathPactException` which (depending on its supervisor) will restart it, and the default `PreRestart` action will terminate all children. Of course this can be handled explicitly as well.

Please note that creating one-off actors from an all-for-one supervisor entails that failures escalated by the temporary actor will affect all the permanent ones. If this is not desired, install an intermediate supervisor; this can very easily be done by declaring a router of size 1 for the worker, see [Routing](xref:routers).

通常阻止孩子（即不响应失败）不会自动终止，其他的孩子都在一个一个的策略；这可以很容易地通过看他们的生命周期：如果`终止`消息不是由主管处理，它将` deathpactexception `这（取决于其主管）将重新启动它，并默认` prerestart `操作会终止所有的孩子。当然，这也可以明确地处理。

请注意，由一对一的主管创建一次性演员意味着临时演员升级失败将影响所有永久演员。如果不需要安装一个中级主管；这可以很容易地被宣布为工人路由器大小1，见[路径]（外部参照：路由器）。
