# Advanced Architecture using ZeroMQ
大规模使用ZeroMQ的影响之一是，由于我们可以比以前更快地构建分布式架构，我们的软件工程流程的局限性变得更加明显。慢动作中的错误往往更难看到（或者说，更容易被合理化）。

我在向工程师群体传授ZeroMQ的经验是，仅仅解释ZeroMQ的工作原理，然后期望他们开始构建成功的产品是远远不够的。像任何消除摩擦的技术一样，ZeroMQ为大的失误打开了大门。如果ZeroMQ是分布式软件开发的ACME火箭推进鞋，那么我们很多人就像Wile E. Coyote一样，全速撞上了传说中的沙漠悬崖。

我们在第6章--ZeroMQ社区中看到，ZeroMQ本身使用了一个正式的流程来进行修改。我们建立这个流程的原因之一，是为了阻止库本身所发生的反复撞崖的情况。

部分原因是为了放慢速度，部分原因是为了确保当你快速行动时，你的方向是正确的--这一点至关重要，亲爱的读者。这是我的标准采访谜语：任何软件系统中最罕见的属性是什么，绝对是最难搞好的东西，缺乏它就会导致绝大多数项目的缓慢或快速死亡？答案不是代码质量、资金、性能，甚至不是（尽管这是一个接近的答案），流行。答案是准确性。

准确性是挑战的一半，适用于任何工程工作。另一半是分布式计算本身，它设置了一系列我们需要解决的问题，如果我们要创建架构。我们需要对数据进行编码和解码；我们需要定义协议来连接客户和服务器；我们需要确保这些协议不受攻击；我们还需要制作健壮的堆栈。异步消息传递是很难做好的。

本章将解决这些挑战，从对如何设计和构建软件的基本重新评估开始，以一个完全成型的大规模文件分发的分布式应用实例结束。

我们将涵盖以下有趣的话题。

* 如何安全地从想法到工作原型（MOPED模式）。
* 将你的数据序列化为ZeroMQ消息的不同方法
* 如何用代码生成二进制序列化的编解码器
* 如何使用GSL工具建立自定义代码生成器
* 如何编写和授权一个协议规范
* 如何在ZeroMQ上建立快速的可重启文件传输
* 如何使用基于信用的流量控制进行非阻塞式传输
* 如何将协议服务器和客户端构建为状态机
* 如何在ZeroMQ上制作一个安全的协议
* 一个大规模的文件发布系统(FileMQ)


##  面向消息的弹性设计模式

我将介绍Message-Oriented Pattern for Elastic Design（MOPED），这是一种用于ZeroMQ架构的软件工程模式。要么是 "MOPED"，要么是 "BIKE"，即Backronym-Induced Kinetic Effect。那是 "BICICLE "的简称，即 "反义词诱导动能效应"（Backronym-Inflated See if I Care Less Effect）。在生活中，人们学会了选择最不难堪的选择。

如果你仔细阅读了这本书，你就已经看到了MOPED的作用。在[#reliable-request-reply]中的Majordomo的发展就是一个近乎完美的案例。但可爱的名字胜过千言万语。

MOPED的目标是定义一个过程，通过这个过程，我们可以为一个新的分布式应用提供一个粗略的用例，并在一周之内用任何语言从 "Hello World "变成完全可行的原型。

使用MOPED，你可以从头开始成长，而不是建立一个有效的ZeroMQ架构，而且失败的风险最小。通过关注契约而不是实现，你可以避免过早优化的风险。通过超短的基于测试的周期来驱动设计过程，你可以更确定你所拥有的东西在你添加更多东西之前是可行的。

我们可以把这变成五个真正的步骤。

* 第一步：将ZeroMQ的语义内部化。
* 第2步：画出一个粗略的架构。
* 第三步：决定合同。
* 第四步：做一个最小的端到端解决方案。
* 第五步：解决一个问题并重复。

###  第一步：内化语义

你必须学习和消化ZeroMQ的 "语言"，也就是套接字模式和它们如何工作。学习一种语言的唯一方法就是使用它。没有办法避免这种投资，没有可以在你睡觉时播放的磁带，没有可以让你插上芯片而神奇地变得更聪明。从一开始就读这本书，用你喜欢的任何语言完成代码实例，了解发生了什么，（最重要的是）自己写一些实例，然后把它们扔掉。

在某一点上，你会感觉到你的大脑里有一种咔咔的声音。也许你会做一个奇怪的由辣椒引起的梦，梦里ZeroMQ的小任务到处跑，想把你活活吃掉。也许你会想 "啊哈，原来*那是*那是什么意思！" 如果我们的工作做得好的话，应该需要两到三天的时间。不管需要多长时间，在你开始用ZeroMQ套接字和模式来思考之前，你还没有准备好进行第二步。

###  第二步：画出一个粗略的架构

根据我的经验，能够画出你的架构的核心是非常重要的。它可以帮助别人理解你的想法，也可以帮助你思考你的想法。要设计一个好的架构，真的没有比用白板向同事解释你的想法更好的方法了。

你不需要把它做对，也不需要把它做完整。你需要做的是把你的架构分成有意义的部分。软件架构的好处是（与建造桥梁相比），如果你把它们隔离开来，你真的可以很便宜地替换整个层。

从选择你要解决的核心问题开始。忽略任何对这个问题不重要的东西：你将在以后把它加进去。这个问题应该是一个端到端的问题：穿过峡谷的绳子。

例如，一个客户要求我们用ZeroMQ做一个超级计算集群。客户端创建工作包，这些工作包被发送到一个代理处，代理处将其分配给工人（在快速图形处理器上运行），收集回结果，并将其返回给客户。

穿过峡谷的绳子是一个客户端与一个经纪人交谈，与一个工人交谈。我们画了三个盒子：客户端、经纪人、工作者。我们在盒子与盒子之间画箭头，显示请求流向一个方向，响应流向另一个方向。这就像我们在前面几章看到的许多图表一样。

要简约。你的目标不是定义一个*真实的*架构，而是在峡谷中抛出一条绳子来引导你的流程。随着时间的推移，我们会使架构成功地变得更加完整和现实：例如，添加多个工作者，添加客户端和工作者API，处理失败，等等。

###  第三步：决定合同的内容

一个好的软件架构取决于合同，合同越明确，事情的规模就越大。你不关心*如何*事情发生；你只关心结果。如果我发送一封电子邮件，我并不关心它是如何到达目的地的，只要合同得到遵守。电子邮件的契约是：它在几分钟内到达，没有人修改它，而且它不会丢失。

而要建立一个运行良好的大型系统，你必须在实施之前关注合同。这听起来很明显，但人们往往忘记或忽视了这一点，或者只是太害羞而不敢强求。我希望我可以说ZeroMQ在这方面做得很好，但多年来，我们的公共契约是二流的事后想法，而不是主要的当面工作。

那么，什么是分布式系统中的契约呢？根据我的经验，有两种类型的合同。

* 客户端应用程序的API。记住心理学要素。API需要尽可能地绝对*简单*，*一致*，和*熟悉*。是的，你可以从代码中生成API文档，但你必须首先设计它，而设计一个API往往是很难的。

* 连接各部分的协议。这听起来像火箭科学，但其实只是一个简单的技巧，而且ZeroMQ让它变得特别容易。事实上，这些协议写起来非常简单，而且不需要太多的官僚主义，我把它们叫做*unprotocols*。

你写最小的契约，这些契约大多只是位置标记。大多数消息和大多数API方法都会丢失或为空。你也要写下任何已知的技术要求，包括吞吐量、延迟、可靠性等等。这些是你将接受或拒绝任何特定工作的标准。

###  第四步：写一个最小的端到端解决方案

目标是尽可能快地测试出整体架构。编写调用API的骨架应用程序，以及实现每个协议两边的骨架堆栈。你想尽快得到一个工作的端到端 "Hello World"。你想在写代码的时候就能测试它，这样你就能剔除那些破碎的假设和不可避免的错误了。不要花六个月的时间去写一个测试套件! 相反，做一个最小的裸露的应用程序，使用我们仍然假设的API。

如果你戴着实现它的人的帽子来设计一个API，你会开始考虑性能、功能、选项等等。你会使它变得更加复杂、更加不规则、更加令人惊讶。但是，这里有一个诀窍（这是个很便宜的诀窍，在日本很流行）：如果你在设计API的时候，戴着必须实际编写使用它的应用程序的人的帽子，你就可以利用所有这些懒惰和恐惧来为你服务。

把协议写在wiki或共享文档上，这样你就可以清楚地解释每一个命令，而不需要太多细节。剥离任何真正的功能，因为它只会产生惯性，让你更难移动东西。你可以随时增加重量。不要花精力去定义正式的消息结构：使用ZeroMQ的多部分框架，以最简单的方式传递最小的信息。

我们的目标是让最简单的测试案例工作，没有任何可避免的功能。所有你能从清单上砍掉的东西，你都要砍掉。不要理会同事和老板的呻吟声。我再重复一次：你可以*总是*增加功能，那是相对容易的。但目标是将整体重量保持在最低水平。

###  第五步：解决一个问题并重复

你现在处于问题驱动开发的快乐循环中，你可以开始解决实际的问题，而不是增加功能。编写问题，每个问题都说明一个明确的问题，并提出一个解决方案。当你设计API的时候，请记住你对名称、一致性和行为的标准。用散文写下这些往往有助于保持他们的理智。

从这里开始，你对架构和代码所做的每一个改变都可以通过运行测试用例来证明，看着它不工作，做出改变，然后看着它工作。

现在，你通过整个周期（扩展测试用例，修复API，更新协议，并根据需要扩展代码），把问题一个一个地解决，并单独测试解决方案。每个周期应该花费大约10-30分钟，偶尔会因为随机的混乱而出现高峰。

##  没有协议的协议

###  没有山羊的协议

当这个人想到协议的时候，这个人想到的是由委员会多年来编写的大量文件。这个人想到了IETF、W3C、ISO、Oasis、监管机构、FRAND专利许可纠纷，不久之后，这个人又想到了退休后去玻利维亚北部山区的一个漂亮的小农场，那里唯一的其他不必要的顽固生物是正在啃食咖啡树的山羊。

现在，我对委员会没有任何个人意见。无用的人需要一个地方来度过他们的生活，而且繁殖的风险最小；毕竟，这似乎才是公平的。但是，大多数委员会的协议倾向于复杂（那些有效的），或者垃圾（那些我们不谈论的）。这有几个原因。一个原因是所涉及的资金量。更多的钱意味着更多的人希望用散文来表达他们特定的偏见和假设。但是，第二个原因是缺乏可以建立的良好的抽象概念。人们曾试图建立可重复使用的协议抽象，如BEEP。大多数人没有坚持下来，而那些坚持下来的，如SOAP和XMPP，则是在事物的复杂方面。

几十年前，当互联网还是一个年轻的小东西时，协议曾经是简短而甜蜜的。它们甚至不是 "标准"，而是 "征求意见"，这是你能得到的最简单的东西。自从我们在1995年创办iMatix以来，我的目标之一就是为像我这样的普通人找到一种方法来编写小而准确的协议，而不需要委员会的开销。

现在，ZeroMQ似乎确实提供了一个活生生的、成功的协议抽象层，它的工作方式是 "我们将在随机传输中携带多部分消息"。因为ZeroMQ默默地处理着框架、连接和路由，所以在ZeroMQ之上编写完整的协议规范是出奇的容易，在[#reliable-request-reply]和[#advanced-pub-sub]中我展示了如何做到这一点。

大约在2007年年中，我启动了数字标准组织，以定义新的更简单的方式来生产小标准、协议和规范。在我看来，那是一个安静的夏天。当时，我写道，一个新的规范应该花[http://www.digistan.org/spec:1 "几分钟来解释，几小时来设计，几天来写，几周来证明，几个月来变得成熟，几年来取代"。］

2010年，我们开始把这样的小规范称为*unprotocols*，有些人可能会误以为这是一个阴暗的国际组织统治世界的卑鄙计划，但它实际上只是意味着 "没有山羊的协议"。

### 合同是困难的

编写合同可能是大规模架构中最困难的部分。通过unprotocols，我们尽可能多地消除了不必要的摩擦。剩下的仍然是一系列难以解决的问题。一个好的合同（无论是API、协议，还是租赁协议）必须简单、明确、技术上合理，并且易于执行。

像任何技术技能一样，这是你必须学习和实践的东西。有一系列的规范在
[http://rfc.zeromq.org ZeroMQ RFC site]，值得一读，当你发现自己有需要时，可以把它们作为你自己的规范的基础。

我试着总结一下我作为一个协议作者的经验。

* 从简单的开始，一步一步地发展你的规范。不要解决你面前没有的问题。

* 使用非常清晰和一致的语言。一个协议可能经常被分解成命令和字段；为这些实体使用清晰的短名称。

* 尽量避免发明概念。从现有的规范中重复使用任何你能做到的东西。使用对你的听众来说明显而清晰的术语。

* 不要做任何你不能证明有迫切需求的东西。你的规范是解决问题的，而不是提供功能的。为你发现的每个问题制定最简单合理的解决方案。

* 实施你的协议*当你建立它时*，这样你就能意识到每个选择的技术后果。使用一种能使它变得困难的语言（如C），而不是一种能使它变得容易的语言（如Python）。

* 在其他人身上测试你的规范*当你构建它时*。当别人在没有你头脑中的假设和知识的情况下试图实现它时，你对规范的最好反馈就是。

* 迅速和持续地进行交叉测试，把别人的客户端和你的服务器进行对比，反之亦然。

* 准备好在需要的时候扔掉并重新开始。通过对你的架构进行分层规划，例如，你可以保留一个API但改变底层协议。

* 只使用独立于编程语言和操作系统的结构。

* 分层解决一个大问题，使每一层都成为独立的规范。谨防创建单一的协议。思考每一层的可重用程度。想一想不同的团队如何在每一层建立相互竞争的规范。

最重要的是，*把它写下来*。代码不是一种规范。关于书面规范的要点是，无论它有多弱，它都可以被系统地改进。通过写下规范，你也会发现代码中不可能看到的不一致和灰色区域。

如果这听起来很难，不要太担心。使用ZeroMQ的一个不太明显的好处是，它将编写协议规范所需的努力减少了90%甚至更多，因为它已经处理了框架、路由、队列等等。这意味着你可以快速地进行实验，廉价地犯错，从而快速地学习。

###  如何编写unprotocols

当你开始写一个unprotocols规范文件时，要坚持一个一致的结构，这样你的读者就知道该期待什么。以下是我使用的结构。

* 封面部分：有1行摘要、规范的URL、正式名称、版本、谁来负责。
* 文本的许可：对于公共规范来说，绝对需要。
* 修改过程：也就是说，作为读者，我怎样才能解决规范中的问题？
* 语言的使用。MUST、MAY、SHOULD等，并参考RFC 2119。
* 成熟度指标：这是一个实验性的、草案的、稳定的、遗留的还是退役的？
* 协议的目标：它试图解决什么问题？
* 正式的语法：防止因对文本的不同解释而产生争论。
* 技术解释：每个消息的语义，错误处理，等等。
* 安全讨论：明确指出，协议的安全性如何。
* 引用：对其他文件、协议等的引用。

写出清晰、有表达力的文字是很难的。要避免试图描述协议的实现。记住，你在写一份合同。你用清晰的语言描述每一方的义务和期望，义务的程度，以及违反规则的惩罚。你并不试图定义*如何*每一方履行其交易的部分。

以下是关于unprotocols的一些关键点。

* 只要你的过程是开放的，那么你就不需要一个委员会：只要做出干净的最小设计，并确保任何人都可以自由地改进它们。

* 如果使用现有的许可证，那么你就不会有事后的法律顾虑。我对我的公共规范使用GPLv3，建议你也这样做。对于内部工作，标准版权是完美的。

* 正式性是有价值的。也就是说，学会写一个正式的语法，如ABNF（Augmented Backus-Naur Form），并使用它来完整地记录你的信息。

* 使用市场驱动的生命周期过程，如[http://www.digistan.org/spec:1 Digistan's COSS]，以便人们在你的规范成熟时（或不成熟时）给予正确的重视。

###  为什么对公共规范使用GPLv3？

你所选择的许可证对于公共规范来说是特别关键的。传统上，协议是在自定义许可证下发布的，作者拥有文本，而衍生作品是被禁止的。这听起来很好（毕竟，谁想看到协议被分叉？），但事实上风险很大。一个协议委员会很容易被捕获，如果该协议很重要，很有价值，那么捕获的动机就会增加。

一旦被抓，就像一些野生动物一样，一个重要的协议往往会死亡。真正的问题是，没有办法*自由*在传统许可下发布的被俘虏的协议。自由 "这个词不仅仅是描述言论或空气的形容词，它也是一个动词，有权利分叉一个作品*违背所有者的意愿*是避免被捕获的关键。

让我用更简短的话来解释一下。想象一下，iMatix今天写了一个协议，这个协议真的很了不起，很受欢迎。我们发布了规范，许多人实施了它。这些实现又快又好，而且像啤酒一样免费。他们开始威胁到一个现有的企业。他们昂贵的商业产品速度较慢，无法与之竞争。所以有一天，他们来到我们位于韩国前塘洞的iMatix办公室，提出要收购我们的公司。因为我们在寿司和啤酒上花费巨大，所以我们感激地接受了。在邪恶的笑声中，协议的新主人停止了对公共版本的改进，关闭了规范，并增加了专利扩展。他们的新产品支持这个新的协议版本，但开放源码版本在法律上被阻止这样做。该公司接管了整个市场，竞争结束。

当你为一个开放源码项目做出贡献时，你真的想知道你的辛勤工作不会被一个封闭源码的竞争者用来对付你。这就是为什么对大多数贡献者来说，GPL胜过 "更宽松 "的BSD/MIT/X11许可证。这些许可证允许作弊。这同样适用于协议和源代码。

当你实施GPLv3规范时，你的应用程序当然是你的，并以任何你喜欢的方式获得许可。但是你可以确定两件事。第一，该规范将*永远不会被接受和扩展为专有形式。该规范的任何衍生形式也必须是GPLv3的。第二，任何实现或使用该协议的人都不会对它所涵盖的任何东西发起专利攻击，他们也不能在不授予世界自由许可的情况下将他们的专利技术添加到协议中。

###  使用ABNF

在编写协议规范时，我的建议是学习和使用正式的语法。这比让别人来解释你的意思，然后从不可避免的错误假设中恢复过来要省事。你的语法的目标是其他人、工程师，而不是编译器。

我最喜欢的语法是ABNF，由[http://www.ietf.org/rfc/rfc2234.txt RFC 2234]定义，因为它可能是定义双向通信协议的最简单和最广泛使用的形式语言。大多数IETF（互联网工程任务组）的规范都使用ABNF，这是个好伙伴。

我将给大家上一堂30秒的ABNF编写速成课。它可能会让你想起正则表达式。你把语法写成规则。每个规则的形式是 "名称=元素"。一个元素可以是另一条规则（你在下面定义为另一条规则），也可以是预先定义的*终端*，如{{CRLF}}、{{OCTET}}，或一个数字。[http://www.ietf.org/rfc/rfc2234.txt The RFC]列出了所有的终端。要定义替代元素，用斜线分开。要定义重复，请使用星号。要对元素进行分组，请使用圆括号。请阅读RFC，因为这并不直观。

我不确定这种扩展是否合适，但我随后用 "C: "和 "S: "作为元素的前缀，以表明它们是来自客户端还是服务器。

这里有一段ABNF，用于一个叫做NOM的unprotocols，我们将在本章的后面再来讨论。

```
Nom-protocol = open-peering *use-peering

open-peering = C:OHAI ( S:OHAI-OK / S:WTF )

open-peering = C:ICANHAZ
                / S:Cheezburger
                / C:HUGZ S:HUGZ-OK
                / S:HUGZ C:HUGZ-OK
```

我实际上已经在商业项目中使用了这些关键词（{{OHAI}}，{{WTF}}）。它们让开发人员傻笑着，很开心。它们使管理层感到困惑。它们在你以后想扔掉的初稿中很好。

###  廉价或讨厌的模式

在几十年的大大小小的协议写作中，我学到了一个普遍的经验。我称之为*廉价或下流*模式：你通常可以把你的工作分成两个方面或层次，分别解决这些问题--一个使用 "廉价 "的方法，另一个使用 "下流 "的方法。

使 "廉价 "或 "下流 "工作的关键见解是要认识到，许多协议混合了用于控制的低容量聊天部分和用于数据的高容量异步部分。例如，HTTP有一个用于认证和获取页面的聊天对话，以及一个用于数据流的异步对话。FTP实际上将其分成了两个端口；一个端口用于控制，一个端口用于数据。

不把控制和数据分开的协议设计者往往会做出可怕的协议，因为这两种情况的权衡几乎是完全相反的。对控制来说是完美的，对数据来说是糟糕的，而对数据来说是理想的，但对控制来说却不适用。当我们想在获得高性能的同时又想获得可扩展性和良好的错误检查时，这一点尤其正确。

让我们用一个经典的客户/服务器用例来分析一下。客户端连接到服务器并进行认证。然后，它要求得到一些资源。服务器回话，然后开始向客户端发送数据。最终，客户端断开连接或服务器结束，对话就结束了。

现在，在开始设计这些信息之前，请停下来想一想，让我们比较一下控制对话和数据流。

* 控制对话持续的时间很短，涉及的消息也很少。数据流可能持续数小时或数天，并涉及数十亿条消息。

* 控制对话框是所有 "正常 "错误发生的地方，例如，未认证、未找到、需要付款、被审查等等。相反，在数据流中发生的任何错误都是特殊的（磁盘满了，服务器崩溃了）。

* 控制对话框是事情会随着时间的推移而改变的地方，因为我们增加了更多的选项、参数等等。数据流几乎不会随着时间的推移而改变，因为资源的语义在一段时间内是相当稳定的。

* 控制对话框本质上是一个同步的请求/回复对话框。数据流本质上是一个单向的异步流。

这些差异是至关重要的。当我们谈论性能时，它只适用于**数据流。把一次性控制对话框设计成快速的，是病态的。因此，当我们谈及序列化的成本时，这只适用于数据流。编码/解码控制流的成本可能是巨大的，而且在许多情况下它不会改变任何东西。所以我们用Cheap来编码控制，用Nasty来编码数据流。

Cheap本质上是同步的、粗略的、描述性的和灵活的。一个Cheap信息充满了丰富的信息，可以为每个应用程序而改变。作为设计者，你的目标是使这些信息易于编码和解析，为实验或增长而进行简单的扩展，并对前后的变化具有高度的稳定性。协议的廉价部分看起来像这样。

* 它使用一个简单的自我描述的数据结构化编码，无论是XML、JSON、HTTP风格的头文件，还是其他。任何编码都可以，只要在你的目标语言中有标准的简单解析器。

* 它使用一个直接的请求-回复模型，每个请求都有一个成功/失败的回复。这使得为一个廉价的对话编写正确的客户端和服务器变得非常简单。

* 它并不试图，甚至是微不足道地，做到快速。当你每个会话只做一次或几次时，性能并不重要。

一个廉价的分析器是你从架子上取下的东西，并把数据扔在上面。它不应该崩溃，不应该泄漏内存，应该有很强的耐受性，而且应该比较容易操作。就是这样。

然而，Nasty本质上是异步的、简洁的、沉默的和不灵活的。Nasty消息携带的信息很少，几乎不会改变。作为设计者，你的目标是使这些信息得到超快的解析，甚至可能无法扩展和实验。理想的Nasty模式是这样的。

* 它使用手工优化的二进制数据布局，其中每一个比特都是精确设计的。

* 它使用一个纯异步模型，其中一个或两个对等体发送数据时没有确认（或者如果他们这样做，他们使用偷偷摸摸的异步技术，如基于信用的流量控制）。


*它没有尝试，哪怕是微不足道的，友好的。当你每秒要做几百万次的事情时，性能是最重要的。

一个讨厌的解析器是你用手写的东西，它单独而精确地写入或读取比特、字节、字和整数。它拒绝任何它不喜欢的东西，完全不做内存分配，而且从不崩溃。

Cheap或Nasty并不是一个普遍的模式；不是所有的协议都有这种二分法。另外，你如何使用Cheap或Nasty将取决于情况。在某些情况下，它可以是一个协议的两个部分。在其他情况下，它可以是两个协议，一个分层在另一个上面。

###  错误处理

使用Cheap或Nasty使错误处理变得相当简单。你有两种命令和两种错误信号的方式。

* 同步控制命令：错误是正常的：每个请求都有一个响应，要么是OK，要么是错误响应。
* 异步数据命令：错误是例外的：坏的命令要么被无声地丢弃，要么导致整个连接被关闭。

通常区分几种错误是很好的，但一如既往地保持最小化，只添加你需要的内容。

##  串行化你的数据

当我们开始设计一个协议时，我们面临的第一个问题是我们如何在电线上对数据进行编码。没有通用的答案。有半打不同的方法来序列化数据，每种方法都有其优点和缺点。我们将探讨其中的一些。

###  抽象级别

在研究如何把数据放到电线上之前，值得问一下我们到底想在应用程序之间交换什么数据。如果我们不使用任何抽象，我们实际上是在序列化和反序列化我们的内部状态。也就是说，我们用来实现我们功能的对象和结构。

然而，将内部状态放在电线上是一个非常糟糕的主意。这就像在一个API中暴露内部状态。当你这样做的时候，你是在把你的实现决定硬编码到你的协议中。你也会产生比它们需要的复杂得多的协议。

这也许是许多老的协议和API如此复杂的主要原因：他们的设计者没有考虑如何把它们抽象成更简单的概念。当然，不能保证抽象会是*更简单*；这就是艰苦工作的地方。

一个好的协议或API抽象封装了自然的使用模式，并赋予它们可预测和有规律的名称和属性。它选择了合理的默认值，以便主要的使用情况可以被最小化。它的目标是对简单的情况要简单，而对较少见的复杂情况要有表达能力。它不对内部实现做任何声明或假设，除非是为了互操作性的绝对需要。

###  ZeroMQ框架

ZeroMQ应用程序最简单和最广泛使用的序列化格式是ZeroMQ自己的多部分框架。例如，下面是[http://rfc.zeromq.org/spec:7 Majordomo Protocol]如何定义一个请求。

```
帧0：空帧
帧1："MDPW01"（六个字节，代表MDP/Worker v0.1)
帧2：0x02（一个字节，代表REQUEST）。
帧3：客户端地址（信封堆栈）。
第4帧：空（零字节，信封定界符）
第5帧以上。请求主体（不透明的二进制）。
```

在代码中阅读和写这个很容易，但这是一个典型的控制流的例子（整个MDP确实是，因为它是一个聊天式的请求-回复协议）。当我们来改进MDP的第二个版本时，我们不得不改变这个框架。很好，我们打破了所有现有的实现!

向后兼容是很难的，但是对控制流使用ZeroMQ的构架*没有帮助*。如果我听从自己的建议，我应该这样设计这个协议（我会在下一个版本中解决这个问题）。它被分成了便宜的部分和讨厌的部分，并使用ZeroMQ的框架来区分这些。

```
第0帧："MDP/2.0 "表示协议名称和版本
第1帧：命令头
第2帧：命令正文
```

我们希望在不同的中间机构（客户端API、经纪人和工作者API）中解析命令头，并在应用程序之间传递未触及的命令体。

###  串行化语言

序列化语言有其时尚性。XML曾经很流行，然后它变得很大，因为它被过度设计了，然后它落入了 "企业信息架构师 "的手中，此后就再也没有人看到它的存在。今天的XML是 "在这个混乱的地方，有一种小型的、优雅的语言正在试图逃脱 "的缩影。

不过，XML还是比它的前辈要好得多，其中包括标准通用标记语言（SGML）这样的怪物，而SGML与EDIFACT这样的折磨人的野兽相比，又是一股清凉之风。因此，序列化语言的历史似乎是逐渐出现的理智，被一波波造反的EIA所掩盖，尽力保住自己的工作。

JSON从JavaScript世界中脱颖而出，作为一种快速和肮脏的 "我宁愿辞职也不愿在这里使用XML "的方式，将数据扔到电线上，然后再拿回来。JSON只是最小的XML，偷偷地表达为JavaScript源代码。

下面是一个在廉价协议中使用JSON的简单例子。

```
"协议"。{
    "name": "MTL",
    "版本": 1
},
"virtual-host": "test-env"
```

同样的数据在XML中会是（XML迫使我们发明一个单一的顶层实体）。

```
<command>（命令
    <协议名称 = "MTL" 版本 = "1" />
    <virtual-host>test-env</virtual-host>。
</command>
```

这里是使用普通的HTTP风格的头文件。

```
协议。MTL/1.0
虚拟主机：test-env
```

只要你不过度使用验证解析器、模式和其他 "相信我们，这都是为了你自己好 "的废话，这些都是相当等同的。廉价的序列化语言为你提供了免费的实验空间（"忽略任何你不认识的元素/属性/头文件"），而且编写通用解析器也很简单，例如，将一个命令分解成一个哈希表，或者反过来。

然而，这不是所有的玫瑰花。虽然现代脚本语言很容易支持JSON和XML，但旧的语言却不支持。如果你使用XML或JSON，你就会产生非同寻常的依赖关系。用C语言处理树状结构的数据也是一种痛苦。

所以你可以根据你的目标语言来驱动你的选择。如果你的世界是一种脚本语言，那么就选择JSON。如果你的目标是为更广泛的系统使用建立协议，那么对C语言的开发者来说要保持简单，坚持使用HTTP风格的头文件。

###  串行化库

{{msgpack.org}}网站说。

> 它就像JSON，但又快又小。MessagePack是一种高效的二进制序列化格式。它可以让你像JSON一样在多种语言之间交换数据，但它更快更小。例如，小的整数（如标志或错误代码）被编码成一个字节，而典型的短字符串只需要在字符串本身之外增加一个字节。

我要提出一个也许不受欢迎的主张，即 "快和小 "是解决非问题的特性。据我所知，序列化库所解决的唯一真正的问题是，需要记录消息契约，并实际将数据序列化到线上和从线下。

让我们先来揭穿 "又快又小 "的说法。它是基于两部分的论点。首先，使你的消息更小，减少编码和解码的CPU成本，将对你的应用程序的性能产生重大影响。第二，这对所有的消息都同样有效。

但是，大多数实际应用往往属于两类中的一类。要么序列化的速度和编码的大小与其他成本相比是微不足道的，如数据库访问或应用程序代码性能。或者，网络性能真的很关键，然后所有重要的成本都发生在几个特定的消息类型中。

因此，全面追求 "快而小 "是一种错误的优化。你既不能为你不经常出现的控制流获得Cheap的轻松灵活性，也不能为你的大批量数据流获得Nasty的残酷效率。更糟的是，假设所有的消息在某种程度上都是平等的，会破坏你的协议设计。Cheap或Nasty不仅仅是关于序列化策略，它也是关于同步与异步、错误处理和变化的成本。

我的经验是，基于消息的应用程序中的大多数性能问题可以通过（a）改进应用程序本身和（b）手工优化高容量的数据流来解决。而要手工优化你最关键的数据流，你需要作弊；学习关于你的数据的利用事实，这是通用序列化程序所不能做到的。

现在让我们来解决文档问题，以及明确和正式地编写我们的合同，而不是只在代码中写。这是一个需要解决的有效问题，如果我们要建立一个持久的、大规模的基于消息的架构，这的确是一个主要问题。

下面是我们如何使用MessagePack接口定义语言（IDL）来描述一个典型的消息。

```
message Person {
  1: string surname
  2: string firstname
  3: optional string email
}
```

现在，使用Google protocol buffers IDL的同一消息。

```
message Person {
  required string surname = 1;
  required string firstname = 2;
  optional string email = 3;
}
```

这很有效，但在大多数实际情况下，比起由手写的或机械生产的体面规范支持的序列化语言，它为你赢得的东西不多（我们将谈到这一点）。你将付出的代价是额外的依赖性，而且很可能比你使用Cheap或Nasty时的整体性能更差。

###  手写的二进制序列化

从这本书中你会发现，我喜欢的系统编程语言是C语言（升级为C99，具有构造器/析构器API模型和通用容器）。我喜欢这种现代化的C语言有两个原因。首先，我的思想太薄弱，无法学习像C## 这样的大语言。生活中似乎充满了更多有趣的东西需要理解。第二，我发现这种特定水平的手动控制使我能够更快地产生更好的结果。

这里的重点不是C与C## ，而是手动控制对高端专业用户的价值。世界上最好的汽车、相机和浓缩咖啡机都有手动控制，这并非偶然。这种现场微调的水平往往使世界一流的成功和次要的成功之间产生差异。

当你真正关心序列化的速度和/或结果的大小时（这些往往是相互矛盾的），你需要手写的二进制序列化。换句话说，让我们听听 "讨厌先生 "的意见吧。

你写一个高效的Nasty编码器/解码器（编解码器）的基本过程是。

* 建立有代表性的数据集和测试应用程序，可以对你的编解码器进行压力测试。
* 编写编解码器的第一个哑巴版本。
* 测试、测量、改进、重复，直到你耗尽时间和/或金钱。

下面是我们用来使我们的编解码器更好的一些技术。

**使用剖析器。*在你对函数计数和每个函数的CPU成本进行剖析之前，根本没有办法知道你的代码在做什么。当你发现你的热点时，就去解决它们。

* 在现代Linux内核上，堆的速度非常快，但它仍然是大多数天真的编解码器的瓶颈。在旧的内核上，堆的速度可能慢得可怜。在你可以的情况下，使用局部变量（堆）而不是堆。

**在不同的平台上，用不同的编译器和编译器选项进行测试。*除了堆之外，还有许多其他的区别。你需要了解主要的差异，并允许它们存在。

* 如果你关心编解码器的性能，你几乎肯定会多次发送相同种类的数据。数据的实例之间会有冗余。你可以检测这些，并利用这些来压缩（例如，一个意味着 "与上次相同 "的短值）。

**了解你的数据。*最好的压缩技术（就压缩的CPU成本而言）需要了解数据的情况。例如，用于压缩一个单词列表、一段视频和一个股市数据流的技术都是不同的。

**准备好打破常规。*你真的需要以大-endian网络字节顺序对整数进行编码吗？x86和ARM几乎占了所有的现代CPU，却使用小-endian（ARM实际上是双-endian，但Android，像Windows和iOS，是小-endian）。

### 代码生成

读了前面两节，你可能会想，"我可以自己写一个比通用的IDL生成器更好的IDL吗？" 如果这个想法在你的脑海中徘徊，它可能很快就离开了，因为它被黑暗中的计算所追赶，不知道这究竟需要多少工作。

如果我告诉你有一种方法可以廉价而快速地建立自定义的IDL生成器呢？你可以有一个方法来获得完美的文档合同，代码是邪恶的和特定领域的，因为你需要它，所有你需要做的是签下你的灵魂（*谁曾经真正使用过，我说得对吗？

在iMatix，直到几年前，我们用代码生成来建立越来越大、越来越雄心勃勃的系统，直到我们认为这项技术（GSL）对于普通人来说太危险了，于是我们封存了档案，用沉重的锁链把它锁在一个深邃的地牢里。我们实际上把它发布在GitHub上。如果你想尝试一下即将出现的例子，请抓取[https://github.com/imatix/gsl the repository]并自己建立一个{{gsl}}命令。在src子目录下输入 "make "应该就可以了（如果你是那个喜欢Windows的人，我相信你会发一个带有项目文件的补丁）。

这一节其实根本不是关于GSL的，而是关于一个有用的、鲜为人知的技巧，它对那些想要扩展自己以及他们的工作的雄心勃勃的建筑师很有用。一旦你学会了这个技巧，你就可以在很短的时间内完成你自己的代码生成器。大多数软件工程师所知道的代码生成器都有一个硬编码的模型。例如，Ragel "从常规语言编译可执行的有限状态机"，也就是说，Ragel的模型是一种常规语言。这对一组好的问题来说当然是有效的，但它远非普遍适用。你如何在Ragel中描述一个API？或者一个项目的Makefile？甚至是一个有限状态机，就像我们在[#可靠的请求-回复]中用来设计二进制之星模式的那个？

所有这些都会从代码生成中受益，但没有通用的模式。所以诀窍是根据你的需要设计你自己的模型，然后把代码生成器作为该模型的廉价编译器。你需要一些关于如何制作好的模型的经验，你还需要一种技术，使建立定制的代码生成器变得便宜。脚本语言，如Perl和Python，是一个不错的选择。然而，我们实际上是专门为此建立了GSL，而这也是我所喜欢的。

让我们举一个简单的例子，与我们已经知道的东西联系起来。我们以后会看到更多的例子，因为我真的相信，代码生成是大规模工作的关键知识。在[#reliable-request-reply]中，我们开发了[http://rfc.zeromq.org/spec:7 Majordomo协议（MDP）]，并为其编写了客户端、经纪人和工人。现在我们能不能通过建立我们自己的接口描述语言和代码生成器来机械地生成这些部分？

当我们写一个GSL模型时，我们可以使用我们喜欢的*任何*语义，换句话说，我们可以在现场发明特定领域的语言。我将发明几个--看看你能不能猜到它们代表什么。

```
slideshow
    name = Cookery level 3
    page
        title = French Cuisine
        item = Overview
        item = The historical cuisine
        item = The nouvelle cuisine
        item = Why the French live longer
    page
        title = Overview
        item = Soups and salads
        item = Le plat principal
        item = Béchamel and other sauces
        item = Pastries, cakes, and quiches
        item = Soufflé: cheese to strawberry
```

这个怎么样：

```
table
    name = person
    column
        name = firstname
        type = string
    column
        name = lastname
        type = string
    column
        name = rating
        type = integer
```

我们可以将第一项编译成一个演示文稿。第二种，我们可以编译成SQL，以创建和处理一个数据库表。所以在这个练习中，我们的领域语言，我们的*模型*，由包含 "信息 "的 "类 "组成，这些信息包含各种类型的 "字段"。这是很熟悉的。这里是MDP客户端协议。

```
<class name = "mdp_client">
    MDP/Client
    <header>
        <field name = "empty" type = "string" value = ""
            >Empty frame</field>
        <field name = "protocol" type = "string" value = "MDPC01"
            >Protocol identifier</field>
    </header>
    <message name = "request">
        Client request to broker
        <field name = "service" type = "string">Service name</field>
        <field name = "body" type = "frame">Request body</field>
    </message>
    <message name = "reply">
        Response back to client
        <field name = "service" type = "string">Service name</field>
        <field name = "body" type = "frame">Response body</field>
    </message>
</class>
```

这是MDP Worker协议：

```
<class name = "mdp_worker">
    MDP/Worker
    <header>
        <field name = "empty" type = "string" value = ""
            >Empty frame</field>
        <field name = "protocol" type = "string" value = "MDPW01"
            >Protocol identifier</field>
        <field name = "id" type = "octet">Message identifier</field>
    </header>
    <message name = "ready" id = "1">
        Worker tells broker it is ready
        <field name = "service" type = "string">Service name</field>
    </message>
    <message name = "request" id = "2">
        Client request to broker
        <field name = "client" type = "frame">Client address</field>
        <field name = "body" type = "frame">Request body</field>
    </message>
    <message name = "reply" id = "3">
        Worker returns reply to broker
        <field name = "client" type = "frame">Client address</field>
        <field name = "body" type = "frame">Request body</field>
    </message>
    <message name = "hearbeat" id = "4">
        Either peer tells the other it's still alive
    </message>
    <message name = "disconnect" id = "5">
        Either peer tells other the party is over
    </message>
</class>
```

GSL使用XML作为其建模语言。XML的名声不好，被拖进了太多的企业下水道，闻起来很甜，但它有一些强有力的积极因素，只要你保持简单。任何编写项目和属性的自我描述层次结构的方法都是可行的。

现在，这里有一个用GSL编写的简短的IDL生成器，可以把我们的协议模型变成文档。

```
.#  Trivial IDL generator (specs.gsl)
.#
.output "$(class.name).md"
## The $(string.trim (class.?''):left) Protocol
.for message
.   frames = count (class->header.field) + count (field)

A $(message.NAME) command consists of a multipart message of $(frames)
frames:

.   for class->header.field
.       if name = "id"
* Frame $(item ()): 0x$(message.id:%02x) (1 byte, $(message.NAME))
.       else
* Frame $(item ()): "$(value:)" ($(string.length ("$(value)")) \
bytes, $(field.:))
.       endif
.   endfor
.   index = count (class->header.field) + 1
.   for field
* Frame $(index): $(field.?'') \
.       if type = "string"
(printable string)
.       elsif type = "frame"
(opaque binary)
.           index += 1
.       else
.           echo "E: unknown field type: $(type)"
.       endif
.       index += 1
.   endfor
.endfor
```

XML模型和此脚本在子目录示例/模型中。 要进行代码生成，我给出此命令：

```
gsl -script:specs mdp_client.xml mdp_worker.xml
```

这是我们为Worker协议获得的Markdown文本：

```
## The MDP/Worker Protocol

A READY command consists of a multipart message of 4 frames:

* Frame 1: "" (0 bytes, Empty frame)
* Frame 2: "MDPW01" (6 bytes, Protocol identifier)
* Frame 3: 0x01 (1 byte, READY)
* Frame 4: Service name (printable string)

A REQUEST command consists of a multipart message of 5 frames:

* Frame 1: "" (0 bytes, Empty frame)
* Frame 2: "MDPW01" (6 bytes, Protocol identifier)
* Frame 3: 0x02 (1 byte, REQUEST)
* Frame 4: Client address (opaque binary)
* Frame 6: Request body (opaque binary)

A REPLY command consists of a multipart message of 5 frames:

* Frame 1: "" (0 bytes, Empty frame)
* Frame 2: "MDPW01" (6 bytes, Protocol identifier)
* Frame 3: 0x03 (1 byte, REPLY)
* Frame 4: Client address (opaque binary)
* Frame 6: Request body (opaque binary)

A HEARBEAT command consists of a multipart message of 3 frames:

* Frame 1: "" (0 bytes, Empty frame)
* Frame 2: "MDPW01" (6 bytes, Protocol identifier)
* Frame 3: 0x04 (1 byte, HEARBEAT)

A DISCONNECT command consists of a multipart message of 3 frames:

* Frame 1: "" (0 bytes, Empty frame)
* Frame 2: "MDPW01" (6 bytes, Protocol identifier)
* Frame 3: 0x05 (1 byte, DISCONNECT)
```

如你所见，这与我在原始规范中手写的内容很接近。现在，如果你已经克隆了{{zguide}}资源库，并且你正在看{{examples/models}}中的代码，你可以生成MDP客户端和工作者编解码。我们将同样的两个模型传递给不同的代码生成器。

```
gsl -script:codec_c mdp_client.xml mdp_worker.xml
```

这给了我们{{mdp_client}}和{{mdp_worker}}类。实际上，MDP是如此简单，以至于几乎不值得为编写代码生成器而付出努力。当我们想改变协议时，利润就来了（我们为独立的Majordomo项目做了这个）。你修改协议，运行命令，就会有更完美的代码出来。

{{codec_c.gsl}}代码生成器并不短，但生成的编解码器要比我最初为Majordomo编写的手写代码好得多。例如，手写的代码没有错误检查，如果你把假信息传给它就会死掉。

我现在要解释一下由GSL驱动的面向模型的代码生成的优点和缺点。权力不是免费的，我们业务中最大的陷阱之一就是凭空发明概念的能力。GSL让这一切变得特别容易，所以它也可能是一个同样危险的工具。

*不要发明概念*。设计师的工作是要消除问题，而不是增加功能。

首先，我将阐述面向模型的代码生成的优势。

* 你可以创建近乎完美的抽象，映射到你的真实世界。因此，我们的协议模型100%地映射到Majordomo的 "真实世界"。如果没有以任何方式调整和改变模型的自由，这将是不可能的。
* 你可以快速而廉价地开发这些完美的模型。
* 你可以生成*任何*文本输出。从一个单一的模型，你可以创建文档，任何语言的代码，测试工具--几乎是你能想到的任何输出。
* 你可以生成（我指的是字面意思）*完美的*输出，因为将你的代码生成器改进到你想要的任何水平很便宜。
* 你可以得到一个结合规范和语义的单一来源。
* 你可以利用一个小的团队，达到一个庞大的规模。在iMatix，我们用大约85K行的输入模型，包括代码生成脚本本身，生产了百万行的OpenAMQ消息产品。

现在让我们来看看它的缺点。

* 你给你的项目增加了工具的依赖性。
* 你可能会得意忘形，为了纯粹的乐趣而创建模型。
* 你可能会使新人疏远你的工作，他们会看到 "奇怪的东西"。
* 你可能会给人们一个强有力的借口，让他们不在你的项目中投资。

愤世嫉俗地说，面向模型的滥用在这样的环境中非常有效：你想产生大量完美的代码，你可以用很少的努力来维护这些代码，而且*没有人可以从你这里拿走。*就我个人而言，我喜欢过河拆桥，继续前进。但如果长期的工作保障是你的事，这几乎是完美的。

因此，如果你真的使用GSL，并想在你的工作周围创建开放的社区，我的建议是： *只在你不使用GSL的地方使用。

* 只在你要用手写令人厌烦的代码的地方使用它。
* 设计人们期望看到的自然模型。
* 首先用手写代码，这样你就知道要生成什么。
* 不要过度使用。保持简单! *不要搞得太玄乎！！*
* 逐步引入到一个项目中。
* 把生成的代码放到你的仓库里。

我们已经在ZeroMQ周围的一些项目中使用GSL。例如，高级的C语言绑定，CZMQ，使用GSL来生成socket选项类（{{zsockopt}}）。一个300行的代码生成器将78行的XML模型变成了1500行完美的、但非常无聊的代码。这是个不错的胜利。

##  传输文件

让我们从说教中休息一下，回到我们的初恋和做这一切的原因：代码。

"我如何发送一个文件？"这是ZeroMQ邮件列表中的一个常见问题。这并不令人惊讶，因为文件传输也许是最古老和最明显的信息传递类型。在网络上发送文件，除了惹恼版权集团之外，还有很多用例。ZeroMQ在发送事件和任务方面开箱即用，但在发送文件方面却不那么出色。

我已经承诺，在一两年的时间里，要写一个适当的解释。这里有一个无偿的信息来照亮你的早晨："适当 "这个词来自于古老的法语*propre*，意思是 "干净"。黑暗时代的英国老百姓不熟悉热水和肥皂，就把这个词改成了 "外国 "或 "上流社会 "的意思，比如 "这就是合适的食物！"，但后来这个词就变成了 "真实 "的意思，比如 "你让我们陷入了一个合适的混乱！"
所以，文件传输。有几个原因，你不能随便拿起一个文件，蒙住它的眼睛，然后把它整个塞进一条信息里。最明显的原因是，尽管数十年来内存大小一直在增长（在我们这些老前辈中，谁不深情地回忆起为1024字节的内存扩展卡所做的储蓄呢！），但磁盘大小仍然顽固地保持着更大。即使我们可以用一条指令发送一个文件（例如，使用像sendfile这样的系统调用），我们也会遇到这样的现实：网络不是无限快，也不是完全可靠。当我们在一个缓慢的摇摆不定的网络上多次尝试上传一个大文件（WiFi，有人吗？也就是说，它需要一种方法，只发送文件中尚未收到的那一部分。

最后，在所有这些之后，如果你建立了一个适当的文件服务器，你会注意到，仅仅是向许多客户发送大量的数据，就会产生我们喜欢称之为 "服务器由于所有可用的堆内存被一个设计不良的应用程序吃掉而倒闭 "的情况，用技术术语来说就是这样。一个适当的文件传输协议需要注意内存的使用。

我们将逐一正确地解决这些问题，这应该有希望让我们在ZeroMQ上运行一个好的、正确的文件传输协议。首先，让我们用随机数据生成一个1GB的测试文件（真正的power-of-two-giga-like-Von-Neumman-intended，而不是内存行业喜欢销售的假硅片）。

```
dd if=/dev/urandom of=testdata bs=1M count=1024
```

当我们有很多客户同时要求获得同一个文件时，这就足够大了，而且在很多机器上，1GB的内存分配是太大了。作为一个基本的参考，让我们测量一下将这个文件从磁盘复制到磁盘上需要多长时间。这将告诉我们我们的文件传输协议在上面增加了多少费用（包括网络费用）。

```
$ time cp testdata testdata2

real    0m7.143s
user    0m0.012s
sys     0m1.188s
```

4位数的精度是误导性的；预计任何一方都有25%的变化。这只是一个 "数量级 "的测量。

这是我们的第一段代码，客户要求提供测试数据，而服务器只是将其作为一系列的消息发送，没有停止呼吸，每条消息都有一个*chunk*。

```c
//  File Transfer model #1
//  
//  In which the server sends the entire file to the client in
//  large chunks with no attempt at flow control.

#include <czmq.h>
#define CHUNK_SIZE  250000

static void
client_thread (void *args, zctx_t *ctx, void *pipe)
{
    void *dealer = zsocket_new (ctx, ZMQ_DEALER);
    zsocket_connect (dealer, "tcp://127.0.0.1:6000");
    
    zstr_send (dealer, "fetch");
    size_t total = 0;       //  Total bytes received
    size_t chunks = 0;      //  Total chunks received
    
    while (true) {
        zframe_t *frame = zframe_recv (dealer);
        if (!frame)
            break;              //  Shutting down, quit
        chunks++;
        size_t size = zframe_size (frame);
        zframe_destroy (&frame);
        total += size;
        if (size == 0)
            break;              //  Whole file received
    }
    printf ("%zd chunks received, %zd bytes\n", chunks, total);
    zstr_send (pipe, "OK");
}

//  .split File server thread
//  The server thread reads the file from disk in chunks, and sends
//  each chunk to the client as a separate message. We only have one
//  test file, so open that once and then serve it out as needed:

static void
server_thread (void *args, zctx_t *ctx, void *pipe)
{
    FILE *file = fopen ("testdata", "r");
    assert (file);

    void *router = zsocket_new (ctx, ZMQ_ROUTER);
    //  Default HWM is 1000, which will drop messages here
    //  because we send more than 1,000 chunks of test data,
    //  so set an infinite HWM as a simple, stupid solution:
    zsocket_set_hwm (router, 0);
    zsocket_bind (router, "tcp://*:6000");
    while (true) {
        //  First frame in each message is the sender identity
        zframe_t *identity = zframe_recv (router);
        if (!identity)
            break;              //  Shutting down, quit
            
        //  Second frame is "fetch" command
        char *command = zstr_recv (router);
        assert (streq (command, "fetch"));
        free (command);

        while (true) {
            byte *data = malloc (CHUNK_SIZE);
            assert (data);
            size_t size = fread (data, 1, CHUNK_SIZE, file);
            zframe_t *chunk = zframe_new (data, size);
            zframe_send (&identity, router, ZFRAME_REUSE + ZFRAME_MORE);
            zframe_send (&chunk, router, 0);
            if (size == 0)
                break;          //  Always end with a zero-size frame
        }
        zframe_destroy (&identity);
    }
    fclose (file);
}

//  .split File main thread
//  The main task starts the client and server threads; it's easier
//  to test this as a single process with threads, than as multiple
//  processes:

int main (void)
{
    //  Start child threads
    zctx_t *ctx = zctx_new ();
    zthread_fork (ctx, server_thread, NULL);
    void *client =
    zthread_fork (ctx, client_thread, NULL);
    //  Loop until client tells us it's done
    char *string = zstr_recv (client);
    free (string);
    //  Kill server thread
    zctx_destroy (&ctx);
    return 0;
}
```

这很简单，但我们已经遇到了一个问题：如果我们向ROUTER套接字发送太多的数据，我们很容易就会溢出。简单但愚蠢的解决方案是在套接字上设置一个无限的高水位线。这很愚蠢，因为我们现在没有保护措施来防止耗尽服务器的内存。然而，如果没有一个无限的HWM，我们就有可能丢失大块的文件。

试试这个：将HWM设置为1000（在ZeroMQ v3.x中这是默认的），然后将块的大小减少到100K，这样我们就可以一次性发送10K的块。运行测试，你会看到它永远不会完成。正如{{zmq_socket[3]}}手册中所说的那样，对于ROUTER套接字来说，有一个愉快的粗暴的说法。"ZMQ_HWM选项动作。Drop"。

我们必须预先控制服务器发送的数据量。如果它发送的数据超过了网络可以处理的范围，那就没有意义了。让我们试着一次发送一个块。在这个版本的协议中，客户端将明确地说，"给我N块"，服务器将从磁盘上获取该特定的块并发送它。

下面是改进后的第二种模式，即客户端一次要求一个块，而服务器只为它从客户端得到的每个请求发送一个块。

```c
//  File Transfer model #2
//  
//  In which the client requests each chunk individually, thus
//  eliminating server queue overflows, but at a cost in speed.

#include <czmq.h>
#define CHUNK_SIZE  250000

static void
client_thread (void *args, zctx_t *ctx, void *pipe)
{
    void *dealer = zsocket_new (ctx, ZMQ_DEALER);
    zsocket_set_hwm (dealer, 1);
    zsocket_connect (dealer, "tcp://127.0.0.1:6000");

    size_t total = 0;       //  Total bytes received
    size_t chunks = 0;      //  Total chunks received

    while (true) {
        //  Ask for next chunk
        zstr_sendm (dealer, "fetch");
        zstr_sendfm (dealer, "%ld", total);
        zstr_sendf (dealer, "%d", CHUNK_SIZE);
        
        zframe_t *chunk = zframe_recv (dealer);
        if (!chunk)
            break;              //  Shutting down, quit
        chunks++;
        size_t size = zframe_size (chunk);
        zframe_destroy (&chunk);
        total += size;
        if (size < CHUNK_SIZE)
            break;              //  Last chunk received; exit
    }
    printf ("%zd chunks received, %zd bytes\n", chunks, total);
    zstr_send (pipe, "OK");
}

//  .split File server thread
//  The server thread waits for a chunk request from a client,
//  reads that chunk, and sends it back to the client:

static void
server_thread (void *args, zctx_t *ctx, void *pipe)
{
    FILE *file = fopen ("testdata", "r");
    assert (file);

    void *router = zsocket_new (ctx, ZMQ_ROUTER);
    zsocket_set_hwm (router, 1);
    zsocket_bind (router, "tcp://*:6000");
    while (true) {
        //  First frame in each message is the sender identity
        zframe_t *identity = zframe_recv (router);
        if (!identity)
            break;              //  Shutting down, quit
            
        //  Second frame is "fetch" command
        char *command = zstr_recv (router);
        assert (streq (command, "fetch"));
        free (command);

        //  Third frame is chunk offset in file
        char *offset_str = zstr_recv (router);
        size_t offset = atoi (offset_str);
        free (offset_str);

        //  Fourth frame is maximum chunk size
        char *chunksz_str = zstr_recv (router);
        size_t chunksz = atoi (chunksz_str);
        free (chunksz_str);

        //  Read chunk of data from file
        fseek (file, offset, SEEK_SET);
        byte *data = malloc (chunksz);
        assert (data);

        //  Send resulting chunk to client
        size_t size = fread (data, 1, chunksz, file);
        zframe_t *chunk = zframe_new (data, size);
        zframe_send (&identity, router, ZFRAME_MORE);
        zframe_send (&chunk, router, 0);
    }
    fclose (file);
}

//  The main task is just the same as in the first model.
//  .skip

int main (void)
{
    //  Start child threads
    zctx_t *ctx = zctx_new ();
    zthread_fork (ctx, server_thread, NULL);
    void *client =
    zthread_fork (ctx, client_thread, NULL);
    //  Loop until client tells us it's done
    char *string = zstr_recv (client);
    free (string);
    //  Kill server thread
    zctx_destroy (&ctx);
    return 0;
}
//  .until
```

现在它要慢得多，因为客户端和服务器之间的来回聊天。在本地环路连接上（客户端和服务器在同一个盒子上），我们为每个请求-回复的往返支付大约300微秒。这听起来不多，但它很快就会增加。

```
$ time ./fileio1
收到4296个块，1073741824字节

真实 0m0.669s
用户 0m0.056s
sys 0m1.048s

$时间 ./fileio2
收到4295个数据块，1073741824字节

真实 0m2.389s
用户 0m0.312s
sys 0m2.136s
```

这里有两个宝贵的经验。首先，虽然请求-回复很容易，但对于大批量的数据流来说，它也太慢了。支付一次300微秒的费用就可以了。为每一个数据块付费是不可接受的，特别是在真正的网络上，其延迟可能要高出1000倍。

第二点是我以前说过的，但要重复一下：在ZeroMQ上实验、测量和改进协议是非常容易的。当某些东西的成本大大降低时，你就可以负担更多的东西。要学会孤立地开发和证明你的协议。我见过一些团队浪费时间去改进设计不良的协议，这些协议深深地嵌入到应用程序中，不容易测试或修复。

除了性能之外，我们的模型二文件传输协议并没有那么糟糕。

* 它完全消除了任何内存耗尽的风险。为了证明这一点，我们把发送方和接收方的高水位都设置为1。
* 它让客户端选择分块大小，这很有用，因为如果要对分块大小进行任何调整，针对网络条件、文件类型，或者进一步减少内存消耗，应该由客户端来做。
* 它为我们提供了完全可重新启动的文件传输。
* 它允许客户端在任何时间点取消文件传输。

如果我们不需要为每个块做一个请求，这将是一个可用的协议。我们需要的是一种方法，让服务器发送多个分块，而不需要等待客户端请求或确认每个分块。我们的选择是什么？

* 服务器可以一次发送10个分块，然后等待一个确认。这就像把块的大小乘以10一样，所以这毫无意义。是的，对于所有10的值来说，这也是毫无意义的。

* 服务器可以在没有来自客户端的任何喋喋不休的情况下发送块，但在每次发送之间有轻微的延迟，这样它就可以在网络可以处理的情况下发送块。这需要服务器知道在网络层发生了什么，这听起来很困难。它还会严重破坏分层。如果网络真的很快，但客户端本身很慢，会发生什么？那时候块状物在哪里排队？

* 服务器可以尝试监视发送队列，也就是说，看看队列有多满，然后只在队列没有满的时候发送。好吧，ZeroMQ不允许这样做，因为它不起作用，原因与节流不起作用相同。服务器和网络可能足够快，但客户端可能是一个缓慢的小设备。

* 我们可以修改{{libzmq}}，以便在到达HWM时采取一些其他行动。也许它可以阻止？这将意味着，一个缓慢的客户端会阻止整个服务器，所以不感谢你。也许它可以向调用者返回一个错误？然后服务器可以做一些聪明的事情，比如......好吧，其实也没有什么能比丢弃信息更好的事情。

除了复杂和各种不愉快之外，这些选择甚至都不可行。我们需要的是一种方法，让客户端在后台异步地告诉服务器，它已经准备好接受更多。我们需要某种异步的流量控制。如果我们做得对，数据应该不间断地从服务器流向客户端，但只有在客户端正在读取数据的时候。让我们回顾一下我们的三个协议。这是第一个。

```
C: fetch
S: chunk 1
S: chunk 2
S: chunk 3
....
```

第二个介绍了每个块的请求：

```
C: fetch chunk 1
S: send chunk 1
C: fetch chunk 2
S: send chunk 2
C: fetch chunk 3
S: send chunk 3
C: fetch chunk 4
....
```

现在 - 波动的手神秘 - 她的一个改变了解决绩效问题的协议：
```
C: fetch chunk 1
C: fetch chunk 2
C: fetch chunk 3
S: send chunk 1
C: fetch chunk 4
S: send chunk 2
S: send chunk 3
....
```

它看起来很可疑地相似。事实上，除了我们发送多个请求而不等待每个请求的回复外，其他都是一样的。这是一种叫做 "流水线 "的技术，它的作用是我们的DEALER和ROUTER套接字是完全异步的。

下面是我们的文件传输测试平台的第三个模型，有管道。客户端提前发送一些请求（"信用"），然后每次处理一个传入的数据块时，都会再发送一个信用。服务器将永远不会发送超过客户要求的块数。

```c
//  File Transfer model #3
//  
//  In which the client requests each chunk individually, using
//  command pipelining to give us a credit-based flow control.

#include <czmq.h>
#define CHUNK_SIZE  250000
#define PIPELINE    10

static void
client_thread (void *args, zctx_t *ctx, void *pipe)
{
    void *dealer = zsocket_new (ctx, ZMQ_DEALER);
    zsocket_connect (dealer, "tcp://127.0.0.1:6000");

    //  Up to this many chunks in transit
    size_t credit = PIPELINE;
    
    size_t total = 0;       //  Total bytes received
    size_t chunks = 0;      //  Total chunks received
    size_t offset = 0;      //  Offset of next chunk request
    
    while (true) {
        while (credit) {
            //  Ask for next chunk
            zstr_sendm  (dealer, "fetch");
            zstr_sendfm (dealer, "%ld", offset);
            zstr_sendf  (dealer, "%ld", (long) CHUNK_SIZE);
            offset += CHUNK_SIZE;
            credit--;
        }
        zframe_t *chunk = zframe_recv (dealer);
        if (!chunk)
            break;              //  Shutting down, quit
        chunks++;
        credit++;
        size_t size = zframe_size (chunk);
        zframe_destroy (&chunk);
        total += size;
        if (size < CHUNK_SIZE)
            break;              //  Last chunk received; exit
    }
    printf ("%zd chunks received, %zd bytes\n", chunks, total);
    zstr_send (pipe, "OK");
}

//  The rest of the code is exactly the same as in model 2, except
//  that we set the HWM on the server's ROUTER socket to PIPELINE
//  to act as a sanity check.
//  .skip

//  The server thread waits for a chunk request from a client,
//  reads that chunk and sends it back to the client:

static void
server_thread (void *args, zctx_t *ctx, void *pipe)
{
    FILE *file = fopen ("testdata", "r");
    assert (file);

    void *router = zsocket_new (ctx, ZMQ_ROUTER);
    //  We have two parts per message so HWM is PIPELINE * 2
    zsocket_set_hwm (router, PIPELINE * 2);
    zsocket_bind (router, "tcp://*:6000");
    while (true) {
        //  First frame in each message is the sender identity
        zframe_t *identity = zframe_recv (router);
        if (!identity)
            break;              //  Shutting down, quit
            
        //  Second frame is "fetch" command
        char *command = zstr_recv (router);
        assert (streq (command, "fetch"));
        free (command);

        //  Third frame is chunk offset in file
        char *offset_str = zstr_recv (router);
        size_t offset = atoi (offset_str);
        free (offset_str);

        //  Fourth frame is maximum chunk size
        char *chunksz_str = zstr_recv (router);
        size_t chunksz = atoi (chunksz_str);
        free (chunksz_str);

        //  Read chunk of data from file
        fseek (file, offset, SEEK_SET);
        byte *data = malloc (chunksz);
        assert (data);

        //  Send resulting chunk to client
        size_t size = fread (data, 1, chunksz, file);
        zframe_t *chunk = zframe_new (data, size);
        zframe_send (&identity, router, ZFRAME_MORE);
        zframe_send (&chunk, router, 0);
    }
    fclose (file);
}

//  The main task starts the client and server threads; it's easier
//  to test this as a single process with threads, than as multiple
//  processes:

int main (void)
{
    //  Start child threads
    zctx_t *ctx = zctx_new ();
    zthread_fork (ctx, server_thread, NULL);
    void *client =
    zthread_fork (ctx, client_thread, NULL);
    //  Loop until client tells us it's done
    char *string = zstr_recv (client);
    free (string);
    //  Kill server thread
    zctx_destroy (&ctx);
    return 0;
}
//  .until
```

这一调整让我们完全控制了端到端的管道，包括所有网络缓冲区和发送方和接收方的ZeroMQ队列。我们确保管道总是充满了数据，同时永远不会超过预定的限制。不仅如此，客户端还能准确决定何时向发送方发送 "信用"。这可能是当它收到一个块，或当它完全处理了一个块。而这是异步发生的，没有明显的性能成本。

在第三个模型中，我选择了一个10条消息的管道规模（每条消息是一个块）。这将花费每个客户端最多2.5MB的内存。所以用1GB的内存，我们至少可以处理400个客户。我们可以试着计算一下理想的管道大小。发送1GB的文件大约需要0.7秒，也就是一个分块大约160微秒。一个来回是300微秒，所以流水线至少需要3-5个块才能让服务器忙起来。在实践中，我仍然得到了5个块的管道的性能峰值，可能是因为信用信息有时会被流出的数据延迟。因此，在10个块的时候，它的工作是稳定的。

```
$ time ./fileio3
收到4291个块，1072741824字节

真实 0m0.777s
用户 0m0.096s
sys 0m1.120s
```

做到严格的测量。你的计算可能是好的，但现实世界往往有自己的意见。

我们所做的显然还不是一个真正的文件传输协议，但它证明了这个模式，我认为它是最简单的合理设计。对于一个真正的工作协议，我们可能想增加一些或全部的内容。

* 认证和访问控制，即使没有加密：重点不是保护敏感数据，而是捕捉错误，如将测试数据发送到生产服务器。

* 一个Cheap式的请求，包括文件路径、可选的压缩和其他我们从HTTP中了解到的有用的东西（比如If-Modified-Since）。

* 一个Cheap风格的响应，至少对于第一个块，提供元数据，如文件大小（所以客户端可以预先分配，并避免不愉快的磁盘满的情况）。

* 一次性获取一组文件的能力，否则协议对大组小文件的效率就会降低。

* 当客户端完全收到一个文件时，客户端要进行确认，以便在客户端意外断开连接时恢复可能丢失的块。

到目前为止，我们的语义一直是 "获取"；也就是说，接收者知道（以某种方式）他们需要一个特定的文件，所以他们要求获取它。然后，关于哪些文件存在以及它们在哪里的知识被带外传递（例如，在HTTP中，通过HTML页面的链接）。

那么，"推送 "语义呢？这方面有两个合理的用例。首先，如果我们采用一种集中式架构，将文件放在一个主要的 "服务器 "上（这不是我所提倡的，但是人们有时确实喜欢这样做），那么允许客户将文件上传到服务器上是非常有用的。其次，它让我们对文件做一种pub-sub的处理，即客户端要求获取所有某种类型的新文件；当服务器得到这些文件时，它将它们转发给客户端。

获取语义是同步的，而推送语义是异步的。异步的不那么健谈，所以速度更快。此外，你还可以做一些可爱的事情，比如 "订阅这个路径"，从而创建一个pub-sub文件传输架构。这显然是非常棒的，我不应该需要解释它解决了什么问题。

不过，这里有一个关于获取语义的问题：告诉客户端存在哪些文件的带外路径。无论你怎么做，最后都会变得很复杂。要么客户端必须进行轮询，要么你需要一个单独的pub-sub通道来保持客户端的最新情况，要么你需要用户互动。

不过，再仔细想想，我们可以看到，fetch只是pub-sub的一个特例。因此，我们可以得到两个世界的最好结果。下面是一般的设计。

* 取回这个路径
* 这里是信用（重复）。

为了使这个工作（我们会的，我亲爱的读者），我们需要更明确地说明我们如何将信用发送到服务器上。把流水线上的 "取块 "请求当作信用的可爱把戏是行不通的，因为客户端不再知道哪些文件实际存在，它们有多大，任何东西。如果客户说，"我可以接受25万字节的数据"，这对1个25万字节的文件或100个2500字节的文件应该同样有效。

这给了我们 "基于信用的流量控制"，它有效地消除了对高水位的需求，以及任何内存溢出的风险。

## 状态机

软件工程师倾向于将（有限）状态机视为一种中介解释器。也就是说，你用一种常规语言将其编译成一个状态机，然后执行状态机。开发者很少能看到状态机本身：它是一种内部表示--优化的、压缩的和奇怪的。

然而，事实证明，状态机作为协议处理程序（如ZeroMQ客户端和服务器）的一流建模语言也很有价值。ZeroMQ让设计协议变得相当容易，但我们从来没有为正确编写这些客户端和服务器定义一个好的模式。

一个协议至少有两个层次。

* 我们如何在电线上表示单个消息。
* 消息如何在对等体之间流动，以及每个消息的意义。

我们在本章中已经看到了如何制作处理序列化的编解码器。这是一个好的开始。但如果我们把第二项工作留给开发者，这就给了他们很大的解释空间。随着我们做出更多雄心勃勃的协议（文件传输+心跳+信用+认证），试图通过手工实现客户端和服务器变得越来越不理智。

是的，人们几乎是系统地这样做的。但是成本很高，而且是可以避免的。我将解释如何使用状态机对协议进行建模，以及如何从这些模型中生成整洁而可靠的代码。

我使用状态机作为软件构建工具的经验可以追溯到1985年，我的第一份真正的工作是为应用程序开发人员制作工具。1991年，我把这些知识变成了一个叫做Libero的免费软件工具，它从一个简单的文本模型中吐出可执行的状态机。

Libero模型的特点是，它是可读的。也就是说，你把你的程序逻辑描述成命名的状态，每个状态接受一组事件，每个状态做一些实际的工作。由此产生的状态机与你的应用程序代码相连接，像老板一样驱动它。

Libero在其工作中表现得非常出色，在许多语言中都很流畅，而且考虑到状态机的神秘性，它的受欢迎程度并不高。我们在几十个大型分布式应用中愤怒地使用了Libero，其中一个在运行了20年后终于在2011年被关闭了。状态机驱动的代码构造工作得非常好，以至于这种方法从未进入软件工程的主流，这让人有些印象深刻。

所以在这一节中，我将解释Libero的模型，并演示如何使用它来生成ZeroMQ客户端和服务器。我们将再次使用GSL，但正如我所说，其原理是通用的，你可以使用任何脚本语言来组装代码生成器。

作为一个工作实例，让我们看看如何在ROUTER套接字上与一个对等体进行有状态的对话。我们将使用一个状态机来开发服务器（而客户端则用手）。我们有一个简单的协议，我称之为 "NOM"。我正在使用哦，非常严肃的[http://unprotocols.org/blog:2 keywords for unprotocols]建议。

```
nom-protocol = open-peering *use-peering

open-peering = C:OHAI ( S:OHAI-OK / S:WTF )

use-peering = C:ICANHAZ
                / S:Cheezburger
                / C:HUGZ S:HUGZ-OK
                / S:HUGZ C:HUGZ-OK
```

我还没有找到一种快速的方法来解释状态机编程的真正性质。根据我的经验，它无一例外地需要几天的实践。在接触了三四天之后，会有一个近乎听得见的 "咔嚓 "声，因为大脑中的某些东西将所有的碎片连接在一起。我们将通过查看我们的NOM服务器的状态机来使其具体化。

状态机的一个有用之处在于，你可以按状态来阅读它们。每个状态都有一个独特的描述性名称和一个或多个*事件*，我们可以按任何顺序列出这些事件。对于每个事件，我们执行0个或更多的*行动*，然后我们移动到一个*下一个状态*（或保持在同一个状态）。

在ZeroMQ协议服务器中，我们有一个状态机实例*每个客户端*。这听起来很复杂，但并不复杂，我们会看到。我们将第一个状态{{Start}}描述为有一个有效事件。{{OHAI}}。我们检查用户的凭证，然后到达认证状态[图]。

```[[ type="textdiagram" title="The Start State"]]
.-------------------.
| Start             |
'-+-----------------'
  |
  #-------------#                    .-------------------.
  | OHAI        +------------------->| Authenticated     |
  #-------------#                    '-------------------'
                   Check Credentials
```

{{Check Credentials}}动作会产生一个{{ok}}或一个{{error}}事件。在认证状态下，我们通过向客户端[图]发送适当的回复来处理这两种可能的事件。如果认证失败，我们就返回到{{Start}}状态，客户可以再试一次。

```[[ type="textdiagram" title="The Authenticated State"]]
.-------------------.
| Authenticated     |
'-+-----------------'
  |
  #-------------#                    .-------------------.
  | ok          +------------------->| Ready             |
  #-------------#                    '-------------------'
  |                Send OHAI-OK
  |
  #-------------#                    .-------------------.
  | error       +------------------->| Start             |
  #-------------#                    '-------------------'
                   Send WTF
```

当认证成功后，我们到达了就绪状态。这里我们有三个可能的事件：一个来自客户端的ICANHAZ或HUGZ消息，或者一个心跳计时器事件[图]。

```[[ type="textdiagram" title="The Ready State"]]
.-------------------.
| Ready             |
'-+-----------------'
  |
  #-------------#                    .-------------------.
  | ICANHAZ     +------------------->| Ready             |
  #-------------#                    '-------------------'
  |                Send CHEEZBURGER
  |
  #-------------#                    .-------------------.
  | HUGZ        +------------------->| Ready             |
  #-------------#                    '-------------------'
  |                Send HUGZ-OK
  |
  #-------------#                    .-------------------.
  | heartbeat   +------------------->| Ready             |
  #-------------#                    '-------------------'
                   Send HUGZ
```
关于这个状态机模型，还有几件事值得了解。

* 大写的事件（如 "HUGZ"）是*外部事件*，来自客户端的消息。
* 小写的事件（如 "heartbeat"）是*内部事件*，由服务器的代码产生。
* "Send SOMETHING "动作是向客户端发送特定回复的简写。
* 没有在特定状态下定义的事件会被默默地忽略。

现在，这些漂亮图片的原始来源是一个XML模型。

```
<class name = "nom_server" script = "server_c">

<state name = "start">
    <event name = "OHAI" next = "authenticated">
        <action name = "check credentials" />
    </event>
</state>

<state name = "authenticated">
    <event name = "ok" next = "ready">
        <action name = "send" message ="OHAI-OK" />
    </event>
    <event name = "error" next = "start">
        <action name = "send" message = "WTF" />
    </event>
</state>

<state name = "ready">
    <event name = "ICANHAZ">
        <action name = "send" message = "CHEEZBURGER" />
    </event>
    <event name = "HUGZ">
        <action name = "send" message = "HUGZ-OK" />
    </event>
    <event name = "heartbeat">
        <action name = "send" message = "HUGZ" />
    </event>
</state>
</class>
```

代码生成器在{{examples/models/server_c.gsl}}中。它是一个相当完整的工具，我将在以后的工作中使用和扩展它。它可以生成。

* 一个C语言的服务器类（{{nom_server.c}}, {{nom_server.h}}），实现了整个协议流程。
* 一个自我测试方法，运行XML文件中列出的自我测试步骤。
* 图形形式的文档（漂亮的图片）。

这里有一个简单的主程序，可以启动生成的NOM服务器。

```c
#include "czmq.h"
#include "nom_server.h"

int main (int argc, char *argv [])
{
    printf ("Starting NOM protocol server on port 5670...\n");
    nom_server_t *server = nom_server_new ();
    nom_server_bind (server, "tcp://*:5670");
    nom_server_wait (server);
    nom_server_destroy (&server);
    return 0;
}
```
生成的nom_server类是一个相当经典的模型。它在ROUTER套接字上接受客户端消息，所以每个请求的第一帧是客户端的连接身份。服务器管理着一组客户，每个客户都有自己的状态。当消息到达时，它将这些消息作为*事件*反馈给状态机。这里是状态机的核心，由GSL命令和我们打算生成的C代码混合而成。

```c
client_execute (client_t *self, int event)
{
    self->next_event = event;
    while (self->next_event) {
        self->event = self->next_event;
        self->next_event = 0;
        switch (self->state) {
.for class.state
            case $(name:c)_state:
.   for event
.       if index () > 1
                else
.       endif
                if (self->event == $(name:c)_event) {
.       for action
.           if name = "send"
                    zmsg_addstr (self->reply, "$(message:)");
.           else
                $(name:c)_action (self);
.           endif
.       endfor
.       if defined (event.next)
                    self->state = $(next:c)_state;
.       endif
                }
.   endfor
                break;
.endfor
        }
        if (zmsg_size (self->reply) > 1) {
            zmsg_send (&self->reply, self->router);
            self->reply = zmsg_new ();
            zmsg_add (self->reply, zframe_dup (self->address));
        }
    }
}
```

每个客户都被当作一个具有各种属性的对象，包括我们需要用来表示一个状态机实例的变量。

```c
event_t next_event;         *  Next event
state_t state;              *  Current state
event_t event;              *  Current event
```

你现在会看到，我们正在生成技术上完美的代码，它具有我们想要的精确设计和形状。说明{{nom_server}}类不是手写的唯一线索是，代码是*太好*。抱怨代码生成器产生糟糕的代码的人，已经习惯了糟糕的代码生成器。根据我们的需要来扩展我们的模型是很容易的。例如，这里是我们如何生成自我测试代码的。

首先，我们在状态机上添加一个 "selftest "项，然后编写我们的测试。我们没有使用任何XML语法或验证，所以这真的只是一个打开编辑器并添加半打文本的问题。

```
<selftest
    <step send = "OHAI" body = "Sleepy" recv = "WTF" />
    <step send = "OHAI" body = "Joe" recv = "OHAI-OK" />
    <step send = "ICANHAZ" recv = "CHEEZBURGER" />
    <step send = "HUGZ" recv = "HUGZ-OK" />
    <step recv = "HUGZ" />
</selftest>
```

在飞行中设计，我决定 "send "和 "recv "是表达 "发送这个请求，然后期待这个回复 "的好方法。下面是将这个模型变成真实代码的GSL代码。

```
.for class->selftest.step
.如果定义了(send)
    msg = zmsg_new ();
    zmsg_addstr (msg, "$(send:)");
. 如果定义了 (body)
    zmsg_addstr (msg, "$(body:)");
. endif
    zmsg_send (&msg, dealer);

. endif
. 如果定义了(recv)
    msg = zmsg_recv (dealer);
    assert (msg);
    command = zmsg_popstr (msg);
    assert (streq (command, "$(recv:)" );
    free (command);
    zmsg_destroy (&msg);

.endif
.endfor
```

最后，任何状态机生成器都有一个比较棘手但绝对必要的部分，那就是*我如何将其插入我自己的代码中？*作为这个练习的最小例子，我想通过接受我的朋友Joe（Hi Joe！）的所有OHAIs并拒绝其他人的OHAIs来实现 "检查证书 "动作。经过思考，我决定直接从状态机模型中抓取代码，即把动作体嵌入到XML文件中。所以在{{nom_server.xml}}中，你会看到这个。

```
<action name = "check credentials">
    char *body = zmsg_popstr (self->request);
    if (body && streq (body, "Joe"))
        self->next_event = ok_event;
    else
        self->next_event = error_event;
    free (body);
</action>
```
而代码生成器会抓取这些C代码，并将其插入生成的{{nom_server.c}}文件中。

```
.for class.action
static void
$(name:c)_action (client_t *self) {
$(string.trim (.):)
}
.endfor
```

现在我们有了一个相当优雅的东西：一个单一的源文件，它描述了我的服务器状态机，同时也包含了我的动作的本地实现。高层和低层的完美结合，比C代码小了大约90%。

请注意，当你的脑袋里旋转着所有你能用这种杠杆产生的惊人的东西的概念时。虽然这种方法给了你真正的力量，但它也使你远离了你的同行，如果你走得太远，你会发现自己在独自工作。

顺便说一下，这个简单的小状态机设计只向我们的自定义代码暴露了三个变量。

* {{self->next_event}}
* {{self->request}}
* {{self->reply}}

在Libero状态机模型中，还有一些我们在这里没有使用的概念，但当我们编写更大的状态机时，我们会需要这些概念。

* 异常，这让我们可以编写更多的状态机。当一个动作引发一个异常时，对该事件的进一步处理就会停止。然后，状态机可以定义如何处理异常事件。
* {{Defaults}}状态，在这里我们可以定义对事件的默认处理（对异常事件特别有用）。

##  使用SASL进行认证

当我们在2007年设计AMQP时，我们选择了[http://en.wikipedia.org/wiki/Simple_Authentication_and_Security_Layer Simple Authentication and Security Layer]（SASL）作为认证层，这是我们从[http://www.rfc-editor.org/rfc/rfc3080.txt BEEP协议框架]中获得的想法之一。SASL初看起来很复杂，但实际上它很简单，而且很适合基于ZeroMQ的协议。我特别喜欢SASL的一点是，它是可扩展的。你可以从匿名访问或纯文本认证和无安全保障开始，并随着时间的推移发展到更安全的机制而不改变你的协议。

我现在不打算做深入的解释，因为我们稍后会看到SASL的行动。但我要解释一下原理，这样你就已经有了一些准备。

在NOM协议中，客户端以OHAI命令开始，服务器要么接受（"Hi Joe！"），要么拒绝。这很简单，但不可扩展，因为服务器和客户端必须事先就他们要做的认证类型达成一致。

SASL引入的是一个完全抽象的、可协商的安全层，在协议层面上仍然很容易实现，这也是一个天才的做法。它的工作方式如下。

* 客户端连接。
* 服务器向客户提出挑战，传递一个它所知道的安全 "机制 "的列表。
* 客户端选择一个它所知道的安全机制，并以一团不透明的数据来回答服务器的挑战，（这里有一个巧妙的技巧）一些通用的安全库计算并提供给客户端。
* 服务器将客户选择的安全机制和这组数据传递给它自己的安全库。
* 该库要么接受客户的答案，要么服务器再次挑战。

有许多免费的SASL库。当我们谈到真正的代码时，我们将只实现两种机制，即ANONYMOUS和PLAIN，它们不需要任何特殊的库。

为了支持SASL，我们必须在我们的 "公开对等 "流程中添加一个可选的挑战/响应步骤。下面是产生的协议语法的样子（我修改了NOM来做这个）。

```
secure-nom      = open-peering *use-peering

open-peering    = C:OHAI *( S:ORLY C:YARLY ) ( S:OHAI-OK / S:WTF )

ORLY            = 1*mechanism challenge
mechanism       = string
challenge       = *OCTET

YARLY           = mechanism response
response        = *OCTET
```

其中ORLY和YARLY包含一个字符串（在ORLY中是一个机制列表，在YARLY中是一个机制）和一个不透明数据的blob。根据机制的不同，来自服务器的初始挑战可能是空的。我们并不关心：我们只是将其传递给安全库来处理。

SASL [http://tools.ietf.org/html/rfc4422 RFC]详细介绍了其他功能（我们不需要），SASL可能被攻击的种类，等等。

##  大规模的文件发布：FileMQ

让我们把所有这些技术整合到一个文件发布系统中，我称之为FileMQ。这将是一个真正的产品，生活在[https://github.com/zeromq/filemq GitHub]上。我们在这里要做的是FileMQ的第一个版本，作为一个培训工具。如果这个概念可行，真实的东西最终可能会有自己的书。

###  为什么要做 FileMQ？

为什么要做一个文件分发系统？我已经解释了如何通过ZeroMQ发送大文件，这真的很简单。但是如果你想让比ZeroMQ多一百万倍的人都能使用信息传递，你就需要另一种API。一个我五岁的儿子可以理解的API。这种API是通用的，不需要编程，并且可以和几乎所有的应用程序一起使用。

是的，我在说文件系统。这是DropBox的模式：把你的文件扔在某个地方，当网络再次连接时，它们会神奇地被复制到其他地方。

然而，我的目标是一个完全去中心化的架构，它看起来更像git，不需要任何云服务（尽管我们可以把FileMQ放在云中），并且可以进行组播，也就是说，可以一次把文件发送到很多地方。

FileMQ必须是安全的（能够），容易与随机脚本语言挂钩，并且在我们的国内和办公网络中尽可能快。

我想用它通过WiFi将照片从我的手机备份到我的笔记本电脑。在一次会议中，在50台笔记本上实时分享演示幻灯片。在会议上与同事分享文件。将地震数据从传感器发送到中央集群。在抗议或骚乱期间，从我的手机上备份视频。在Linux服务器的云端同步配置文件。

一个有远见的想法，不是吗？嗯，想法很便宜。困难的部分是使这个，并且使它简单。

###  最初的设计切割：API

这里是我看到的第一个设计的方式。FileMQ必须是分布式的，这意味着每个节点可以同时是服务器和客户端。但我不希望协议是对称的，因为那看起来很勉强。我们有一个文件从A点到B点的自然流动，其中A是 "服务器"，B是 "客户端"。如果文件从另一个方向流回来，那么我们就有两个流。FileMQ还不是目录同步协议，但我们将使它相当接近。

因此，我将把FileMQ构建为两块：一个客户端和一个服务器。然后，我将把这些放在一个主程序中（{{filemq}}工具），它既可以作为客户端也可以作为服务器。这两块东西看起来会和{{nom_server}}很相似，有相同的API。

```c
fmq_server_t *server = fmq_server_new ()。
fmq_server_bind (server, "tcp://*:5670")。
fmq_server_publish (server, "/home/ph/filemq/share", "/public")。
fmq_server_publish (server, "/home/ph/photos/stream", "/photostream")。

fmq_client_t *client = fmq_client_new ();
fmq_client_connect (client, "tcp://pieter.filemq.org:5670");
fmq_client_subscribe (server, "/public/", "/home/ph/filemq/share")。
fmq_client_subscribe (server, "/photostream/", "/home/ph/photos/stream")。

while (!zctx_interrupted)
    sleep (1);

fmq_server_destroy（&server）。
fmq_client_destroy (&client);
```

如果我们用其他语言来包装这个C语言的API，我们就可以很容易地对FileMQ进行编程，嵌入它的应用程序，把它移植到智能手机上，等等。

###  初始设计切割：协议

该协议的全称是 "文件消息队列协议"，或大写的FILEMQ，以区别于软件。首先，我们把协议写成一个ABNF语法。我们的语法从客户端和服务器之间的命令流开始。你应该认识到这些是我们已经看到的各种技术的组合。

```
filemq-protocol = open-peering *use-peering [ close-peering ]

open-peering    = C:OHAI *( S:ORLY C:YARLY ) ( S:OHAI-OK / error )

use-peering     = C:ICANHAZ ( S:ICANHAZ-OK / error )
                / C:NOM
                / S:CHEEZBURGER
                / C:HUGZ S:HUGZ-OK
                / S:HUGZ C:HUGZ-OK

close-peering   = C:KTHXBAI / S:KTHXBAI

error           = S:SRSLY / S:RTFM
```

这是往返服务器的命令：

```
;   The client opens peering to the server
OHAI            = signature %x01 protocol version
signature       = %xAA %xA3
protocol        = string        ; Must be "FILEMQ"
string          = size *VCHAR
size            = OCTET
version         = %x01

;   The server challenges the client using the SASL model
ORLY            = signature %x02 mechanisms challenge
mechanisms      = size 1*mechanism
mechanism       = string
challenge       = *OCTET        ; ZeroMQ frame

;   The client responds with SASL authentication information
YARLY           = %signature x03 mechanism response
response        = *OCTET        ; ZeroMQ frame

;   The server grants the client access
OHAI-OK         = signature %x04

;   The client subscribes to a virtual path
ICANHAZ         = signature %x05 path options cache
path            = string        ; Full path or path prefix
options         = dictionary
dictionary      = size *key-value
key-value       = string        ; Formatted as name=value
cache           = dictionary    ; File SHA-1 signatures

;   The server confirms the subscription
ICANHAZ-OK      = signature %x06

;   The client sends credit to the server
NOM             = signature %x07 credit
credit          = 8OCTET        ; 64-bit integer, network order
sequence        = 8OCTET        ; 64-bit integer, network order

;   The server sends a chunk of file data
CHEEZBURGER     = signature %x08 sequence operation filename
                  offset headers chunk
sequence        = 8OCTET        ; 64-bit integer, network order
operation       = OCTET
filename        = string
offset          = 8OCTET        ; 64-bit integer, network order
headers         = dictionary
chunk           = FRAME

;   Client or server sends a heartbeat
HUGZ            = signature %x09

;   Client or server responds to a heartbeat
HUGZ-OK         = signature %x0A

;   Client closes the peering
KTHXBAI         = signature %x0B
```

这是服务器可以告诉客户端出现问题的不同方式：

```
;   Server error reply - refused due to access rights
S:SRSLY         = signature %x80 reason

;   Server error reply - client sent an invalid command
S:RTFM          = signature %x81 reason
```

FILEMQ 住在 [http://rfc.zeromq.org/spec:19 ZeroMQ unprotocols website] 上，并在 IANA（Internet Assigned Numbers Authority）注册了一个 TCP 端口，即 5670 端口。

###  构建和尝试 FileMQ

FileMQ栈是[https://github.com/zeromq/filemq on GitHub]。它像一个经典的C/C## 项目一样工作。

```
git clone git://github.com/zeromq/filemq.git
cd filemq
./autogen.sh
./configure
make check
```

你希望使用最新的CZMQ主程序来做这个。现在试着运行{{track}}命令，这是一个简单的工具，使用FileMQ来跟踪一个目录在另一个目录中的变化。

```
cd src
./track ./fmqroot/send ./fmqroot/recv
```

并打开两个文件导航窗口，一个进入{{src/fmqroot/send}}，一个进入{{src/fmqroot/recv}}。将文件放入发送文件夹，你会看到它们到达recv文件夹。服务器每秒钟检查一次新文件。删除send文件夹中的文件，它们也会同样在recv文件夹中被删除。

我使用轨道来更新我的MP3播放器，作为一个USB驱动器安装。当我在笔记本电脑的音乐文件夹中添加或删除文件时，MP3播放器上也会发生同样的变化。FILEMQ还不是一个完整的复制协议，但我们以后会解决这个问题。

### 内部架构

为了构建FileMQ，我使用了大量的代码生成器，对于一个教程来说可能太多了。然而，这些代码生成器在其他堆栈中都是可以重用的，对我们在[#moving-pieces]中的最终项目也很重要。它们是由我们之前看到的那一套演变而来的。

* {{codec_c.gsl}}：为给定的协议生成一个消息编解码器。
* {{server_c.gsl}}：为一个协议和状态机生成一个服务器类。
* {{client_c.gsl}}：为协议和状态机生成一个客户端类。

学习使用GSL代码生成的最好方法是将这些代码翻译成你选择的语言，并制作你自己的演示协议和堆栈。你会发现这相当容易。FileMQ本身并不试图支持多语言。它可以，但它会使事情变得不必要地复杂。

FileMQ的架构实际上分为两层。有一套通用的类来处理分块、目录、文件、补丁、SASL安全和配置文件。然后，是生成的堆栈：消息、客户端和服务器。如果我正在创建一个新的项目，我会分叉整个FileMQ项目，然后去修改这三个模型。

* {{fmq_msg.xml}}: 定义消息格式
* {{fmq_client.xml}}：定义了客户端状态机、API和实现。
* {{fmq_server.xml}}：为服务器做同样的事情。

你会想把东西重新命名，以避免混淆。为什么我不把可重用的类做成一个单独的库？答案是两方面的。首先，没有人真正需要这个（还没有）。第二，当你构建和使用 FileMQ 时，它会使事情变得更加复杂。为了解决一个理论上的问题而增加复杂性是不值得的。

虽然我用C语言写了FileMQ，但它很容易映射到其他语言。当你加入CZMQ的通用zlist和zhash容器和类的风格时，C语言变得如此之好，是相当惊人的。让我快速浏览一下这些类。

* {{fmq_sasl}}：对SASL挑战进行编码和解码。我只实现了PLAIN机制，这足以证明这个概念。

* {{fmq_chunk}}：用于处理大小不一的Blobs。不像ZeroMQ的消息那样高效，但它们做的怪事较少，所以更容易理解。块类有从磁盘上读取和写入块的方法。

* {{fmq_file}}：与文件一起工作，这些文件可能存在于磁盘上，也可能不存在。给你提供文件的信息（如大小），让你读写文件，删除文件，检查文件是否存在，以及检查文件是否 "稳定"（后面会详细介绍）。

* {{fmq_dir}}：处理目录，从磁盘上读取目录，比较两个目录，看有什么变化。如果有变化，则返回一个 "补丁 "的列表。

* {{fmq_patch}}：处理一个补丁，实际上只是说 "创建这个文件 "或 "删除这个文件"（每次都指一个fmq_file项目）。

* {{fmq_config}}：与配置数据一起工作。我以后会再来讨论客户端和服务器的配置。

每个类都有一个测试方法，主要的开发周期是 "编辑、测试"。这些大多是简单的自我测试，但它们使得我可以信任的代码和我知道仍然会损坏的代码之间的区别。可以肯定的是，任何没有被测试案例覆盖的代码都会有未被发现的错误。我不是一个外部测试线束的粉丝。但是你在写功能时写的内部测试代码......那就像刀上的把手。

你应该，真的，能够阅读源代码并迅速理解这些类在做什么。如果你不能愉快地阅读代码，请告诉我。如果你想把FileMQ的实现移植到其他语言中，可以从分叉整个仓库开始，以后我们会看看是否有可能在一个整体仓库中完成。

### 公共API

公共API由两个类组成（如我们前面的草图）。

* {{fmq_client}}：提供客户端API，包括连接到服务器、配置客户端和订阅路径的方法。

* {{fmq_server}}：提供服务器API，包括绑定端口、配置服务器和发布路径的方法。

这些类提供了一个*多线程的API*，这种模式我们已经用过好几次了。当你创建一个API实例时（即{{fmq_server_new()}}或{{fmq_client_new()}}），这个方法会启动一个后台线程来做真正的工作，即运行服务器或客户端。然后，其他API方法通过ZeroMQ套接字（由inproc://上的两个PAIR套接字组成的*管道*）与这个线程对话。

如果我是一个热衷于在另一种语言中使用FileMQ的年轻开发者，我可能会花一个愉快的周末为这个公共API写一个绑定，然后把它放在filemq项目的一个子目录下，比如说，{{bindings/}}，并提出一个拉取请求。

实际的API方法来自于状态机的描述，像这样（针对服务器）。

```
<method name = "publish">
<argument name = "location" type = "string" />
<argument name = "alias" type = "string" />
mount_t *mount = mount_new (location, alias);
zlist_append (self->mounts, mount);
</method>
```

这被变成了这段代码。

```c
void
fmq_server_publish (fmq_server_t *self, char *location, char *alias)
{
    assert (self);
    assert (location);
    assert (alias);
    zstr_sendm (self->pipe, "PUBLISH");
    zstr_sendfm (self->pipe, "%s", location);
    zstr_sendf (self->pipe, "%s", alias);
}
```

###  设计说明

制作FileMQ最难的部分并不是实现协议，而是在内部保持准确的状态。一个FTP或HTTP服务器本质上是无状态的。但是一个发布/订阅服务器*必须*维护订阅，至少是这样。

所以我将通过一些设计方面的内容。

* 客户端通过缺少来自服务器的心跳（{{HUGZ}}）来检测服务器是否已经死亡。然后，它通过发送一个{{OHAI}}来重新启动对话。{{OHAI}}没有超时，因为ZeroMQ DEALER套接字会无限期地排队发送消息。

* 如果客户端不再对服务器发送的心跳做出回复（{{HUGZ-OK}}），服务器就会认为该客户端已经死亡，并删除该客户端的所有状态，包括其订阅。

* 客户端API将订阅保存在内存中，并在成功连接后重放它们。这意味着调用者可以在任何时候订阅（并且不关心连接和认证何时实际发生）。

* 服务器和客户端使用虚拟路径，很像一个HTTP或FTP服务器。你发布一个或多个*mount point*，每个都对应于服务器上的一个目录。每一个都映射到一些虚拟路径，比如说"/"，如果你只有一个挂载点。客户端然后订阅虚拟路径，文件到达收件箱目录中。我们不在网络上发送物理文件名。

* 有一些时间问题：如果服务器正在创建它的挂载点，而客户正在连接和订阅，那么订阅将不会附加到正确的挂载点。所以，我们最后才绑定服务器的端口。

* 客户端可以在任何时候重新连接；如果客户端发送{{OHAI}}，那就表示之前的对话结束，新的对话开始。我可能有一天会让订阅在服务器上持久化，这样它们就能在断开连接后继续存在。客户端堆栈在重新连接后，会重放调用者应用程序已经进行的任何订阅。

###  配置

我已经建立了几个大型的服务器产品，比如90年代末流行的Xitami网络服务器，以及[http://www.openamq.org OpenAMQ消息服务器]。让配置变得简单和明显是使这些服务器使用起来很有趣的一个重要原因。

我们通常旨在解决一些问题。

* 与产品一起运送默认的配置文件。
* 允许用户添加自定义的配置文件，这些文件不会被覆盖。
* 允许用户从命令行进行配置。

然后把这些东西分层，让命令行设置覆盖自定义设置，而自定义设置又覆盖默认设置。要做到这一点，可能需要很多工作。对于FileMQ，我采取了一种更简单的方法：所有的配置都从API中完成。

例如，这就是我们启动和配置服务器的方法。

```c
server = fmq_server_new ();
fmq_server_configure（server，"server_test.cfg"）。
fmq_server_publish (server, "./fmqroot/send", "/")。
fmq_server_publish (server, "./fmqroot/logs", "/logs");
fmq_server_bind (server, "tcp://*:5670");
```

我们确实为配置文件使用了一种特定的格式，即[http://rfc.zeromq.org/spec:4 ZPL]，这是一种极简的语法，我们几年前就开始为ZeroMQ "设备 "使用，但对任何服务器都很适用。

```ini
#   Configure server for plain access
#
server
    monitor = 1             #   Check mount points
    heartbeat = 1           #   Heartbeat to clients

publish
    location = ./fmqroot/logs
    virtual = /logs

security
    echo = I: use guest/guest to login to server
    #   These are SASL mechanisms we accept
    anonymous = 0
    plain = 1
        account
            login = guest
            password = guest
            group = guest
        account
            login = super
            password = secret
            group = admin
```

生成的服务器代码做的一件可爱的事情（似乎很有用）是解析这个配置文件（当你使用{{fmq_server_configure()}}方法时），并执行任何与API方法匹配的部分。因此，{{publish}}部分作为{{fmq_server_publish()}}方法工作。

### 文件的稳定性

轮询一个目录的变化，然后对新文件做一些 "有趣 "的事情，这是非常常见的。但由于一个进程正在向一个文件写入，其他进程不知道该文件何时被完全写入。一个解决方案是，在创建第一个文件之后，我们再添加一个 "指示器 "文件。然而，这是很麻烦的。

有一个更简洁的方法，那就是检测一个文件是否 "稳定"，也就是说，没有人再向它写东西。FileMQ通过检查文件的修改时间来做到这一点。如果它的时间超过一秒，那么该文件就被认为是稳定的，至少稳定到可以被运送到客户端。如果五分钟后有一个进程出现并追加了文件，那么它就会被再次发送出去。

为了使其发挥作用，这也是任何希望成功使用FileMQ的应用程序的要求，在写之前，不要在内存中缓冲超过一秒钟的数据。如果你使用非常大的块大小，文件可能看起来很稳定，其实不然。

### 交付通知

我们使用的多线程API模型的一个好处是，它基本上是基于消息的。这使得它非常适合于将事件返回给调用者。一个更传统的API方法是使用回调。但是跨越线程边界的回调有些微妙。下面是当客户端收到一个完整的文件时，它是如何发回一条消息的。

```c
zstr_sendm (self->pipe, "DELIVER");
zstr_sendm (self->pipe, filename);
zstr_sendf (self->pipe, "%s/%s", inbox, filename);
```

我们现在可以给API添加一个_recv()方法，等待从客户端返回的事件。它为调用者做出了一个简洁的风格：创建客户端对象，配置它，然后接收和处理它返回的任何事件。

###  符号链接

虽然使用暂存区是一个不错的、简单的API，但它也为发送者创造了成本。如果我在相机上已经有了一个2GB的视频文件，并想通过FileMQ发送它，目前的实现要求我在将其发送给订阅者之前将其复制到一个暂存区域。

一种选择是挂载整个内容目录（例如，{{/home/me/Movies}}），但这是脆弱的，因为这意味着应用程序不能决定发送单个文件。要不就是一切，要不就是一无所有。

一个简单的答案是实现便携式符号链接。正如维基百科所解释的。"一个符号链接包含一个文本字符串，它被操作系统自动解释并作为另一个文件或目录的路径来遵循。这个其他文件或目录被称为*目标*。符号链接是一个独立于其目标而存在的第二个文件。如果一个符号链接被删除，其目标仍然不受影响"。

这不会以任何方式影响协议；它是服务器实现中的一个优化。让我们做一个简单的可移植实现。

* 一个符号链接由一个扩展名为{{.ln}}的文件组成。
* 没有{{.ln}}的文件名是发布的文件名。
* 该链接文件包含一行，是文件的真实路径。

因为我们把对文件的所有操作都收集在一个类中（{{fmq_file}}），所以这是一个干净的变化。当我们创建一个新的文件对象时，我们会检查它是否是一个符号链接，然后所有的只读操作（获取文件大小，读取文件）都在目标文件上操作，而不是链接。

### 恢复和晚期加入者

就目前而言，FileMQ还有一个主要问题：它没有为客户端提供从故障中恢复的方法。这种情况是，一个连接到服务器的客户端开始接收文件，然后由于某种原因断开连接。网络可能太慢，或者断裂。客户端可能在一台笔记本电脑上，它被关闭了，然后又恢复了。WiFi可能被断开了。随着我们进入一个更加移动的世界（见[#moving-pieces]），这种用例变得越来越频繁。在某些方面，它正成为一个主导的用例。

在经典的ZeroMQ pub-sub模式中，有两个强有力的基本假设，这两个假设在FileMQ的现实世界中通常是错误的。首先，数据过期非常快，所以没有兴趣从旧数据中询问。第二，网络是稳定的，很少发生故障（所以最好在改善基础设施方面多投资，而在解决恢复方面少投资）。

就拿任何FileMQ的用例来说，你会发现，如果客户端断开连接并重新连接，那么它应该得到任何它错过的东西。进一步的改进是要从部分故障中恢复，就像HTTP和FTP那样。但一次只能做一件事。

恢复的一个答案是 "持久订阅"，FILEMQ协议的第一份草案旨在支持这一点，服务器可以保留和存储客户端的标识。因此，如果一个客户端在故障后重新出现，服务器将知道它没有收到哪些文件。

然而，有状态的服务器制作起来很麻烦，而且很难扩展。例如，我们如何进行故障转移到第二台服务器？它从哪里得到它的订阅？如果每个客户端连接都独立工作并携带所有必要的状态，那就好得多了。

持久订阅的另一个问题是，它需要前期的协调。前期协调总是一个红旗，不管是在一个人的团队中，还是在一堆流程的相互交谈中。迟到的加入者怎么办？在现实世界中，客户不会整齐地排队，然后同时说 "准备好了！"。在现实世界中，他们来去自如，如果我们能以对待一个全新的客户和一个离开后又回来的客户一样的方式来对待，那就很有价值了。

为了解决这个问题，我将在协议中加入两个概念：一个*resynchronization*选项和一个{{cache}}字段（一个字典）。如果客户端想要恢复，它就设置重新同步选项，并通过{{cache}}字段告诉服务器它已经有哪些文件。我们需要这两者，因为协议中没有办法区分空字段和空字段。FILEMQ RFC对这些字段的描述如下。

> {{options}}字段为服务器提供额外信息。服务器应实现这些选项。{{RESYNC=1}}。- 如果客户端设置了该选项，服务器应向客户端发送虚拟路径的全部内容，但客户端已经拥有的文件除外，这些文件由{{cache}}字段中的SHA-1摘要标识。

还有。

> 当客户端指定{{RESYNC}}选项时，{{cache}}字典字段会告诉服务器客户端已经拥有哪些文件。{{cache}}字典中的每个条目是一个 "文件名=摘要 "的键/值对，其中摘要必须是可打印的十六进制SHA-1摘要。如果文件名以"/"开头，那么它应该以路径开始，否则服务器必须忽略它。如果文件名不是以"/"开头，那么服务器应将其视为相对于路径的文件名。

知道自己处于经典的pub-sub用例的客户不提供任何缓存数据，而想要恢复的客户则提供他们的缓存数据。这不需要服务器的状态，也不需要前期的协调，对于全新的客户端（可能是通过一些带外的方式收到了文件），以及收到一些文件后断开了一段时间的客户端，都同样适用。

我决定使用SHA-1摘要有几个原因。首先，它足够快：在我的笔记本电脑上，150毫秒就可以消化一个25MB的核心转储。第二，它很可靠：一个文件的不同版本得到相同的哈希值的机会接近于零。第三，它是最广泛支持的摘要算法。循环冗余检查（如CRC-32）更快，但不可靠。最近的SHA版本（SHA-256，SHA-512）更安全，但要多花50%的CPU周期，对我们的需求来说是多余的。

下面是当我们同时使用缓存和重同步时，一个典型的ICANHAZ消息的样子（这是生成的编解码类的{{dump}}方法的输出）。

```
ICANHAZ:
    path='/photos'
    options={
        RESYNC=1
    }
    cache={
        DSCF0001.jpg=1FABCD4259140ACA99E991E7ADD2034AC57D341D
        DSCF0006.jpg=01267C7641C5A22F2F4B0174FFB0C94DC59866F6
        DSCF0005.jpg=698E88C05B5C280E75C055444227FEA6FB60E564
        DSCF0004.jpg=F0149101DD6FEC13238E6FD9CA2F2AC62829CBD0
        DSCF0003.jpg=4A49F25E2030B60134F109ABD0AD9642C8577441
        DSCF0002.jpg=F84E4D69D854D4BF94B5873132F9892C8B5FA94E
    }
```

虽然我们在FileMQ中没有这样做，但服务器可以使用缓存信息来帮助客户端赶上它所错过的删除。要做到这一点，它必须记录删除，然后在客户端订阅时将此日志与客户端的缓存进行比较。

###  测试用例：跟踪工具

为了正确地测试像FileMQ这样的东西，我们需要一个测试用例来处理实时数据。我的系统管理任务之一是管理我的音乐播放器上的MP3音轨，顺便说一下，这是一个用Rock Box刷新的Sansa Clip，我强烈推荐它。当我把曲目下载到我的音乐文件夹时，我想把这些曲目复制到我的播放器上，当我发现那些令我讨厌的曲目时，我就在音乐文件夹中删除它们，并希望这些曲目也从我的播放器中消失。

对于一个强大的文件分发协议来说，这有点矫枉过正。我可以用bash或Perl脚本来写这个，但说实话，FileMQ中最难的工作是目录比较代码，我想从中受益。所以我组装了一个叫{{track}}的简单工具，它调用FileMQ的API。在命令行中，这个工具运行时有两个参数；发送目录和接收目录。

```
./track /home/ph/Music /media/3230-6364/MUSIC
```

这段代码是一个很好的例子，说明如何使用FileMQ API来做本地文件分发。这里是完整的程序，去掉了许可文本（它是MIT/X11许可）。

```
#include "czmq.h"
#include "../include/fmq.h"

int main (int argc, char *argv [])
{
    fmq_server_t *server = fmq_server_new ();
    fmq_server_configure (server, "anonymous.cfg");
    fmq_server_publish (server, argv [1], "/");
    fmq_server_set_anonymous (server, true);
    fmq_server_bind (server, "tcp://*:5670");

    fmq_client_t *client = fmq_client_new ();
    fmq_client_connect (client, "tcp://localhost:5670");
    fmq_client_set_inbox (client, argv [2]);
    fmq_client_set_resync (client, true);
    fmq_client_subscribe (client, "/");

    while (true) {
        *  Get message from fmq_client API
        zmsg_t *msg = fmq_client_recv (client);
        if (!msg)
            break;              *  Interrupted
        char *command = zmsg_popstr (msg);
        if (streq (command, "DELIVER")) {
            char *filename = zmsg_popstr (msg);
            char *fullname = zmsg_popstr (msg);
            printf ("I: received %s (%s)\n", filename, fullname);
            free (filename);
            free (fullname);
        }
        free (command);
        zmsg_destroy (&msg);
    }
    fmq_server_destroy (&server);
    fmq_client_destroy (&client);
    return 0;
}
```

注意我们在这个工具中是如何处理物理路径的。服务器发布物理路径{{/home/ph/Music}}并将其映射到虚拟路径{{/}}。客户端订阅{{/}}，接收{{/media/3230-6364/MUSIC}}中的所有文件。我可以使用服务器目录内的任何结构，它将被忠实地复制到客户端的收件箱中。注意API方法{{fmq_client_set_resync()}}，它引起服务器到客户端的同步。

##  获得一个官方的端口号

在FILEMQ的例子中，我们一直在使用5670端口。与本书之前的所有例子不同，这个端口不是任意的，而是由互联网[http://www.iana.org Assigned Numbers Authority (IANA)]分配的，它 "负责全球协调DNS根、IP寻址和其他互联网协议资源"。

我将非常简要地解释何时以及如何为你的应用协议申请注册端口号。主要原因是确保你的应用程序可以在野外运行而不与其他协议冲突。从技术上讲，如果你运送的任何软件使用1024到49151之间的端口号，你应该只使用IANA注册的端口号。然而，许多产品并不理会这一点，而是倾向于使用IANA的列表作为 "避免使用的端口"。

如果你的目标是做一个重要的公共协议，比如FILEMQ，你将会想要一个IANA注册的端口。我将简要地解释如何做到这一点。

* 清楚地记录你的协议，因为IANA会要求你说明你打算如何使用这个端口。它不必是一个完整的协议规范，但必须足够坚实，以通过专家审查。

* 决定你想要什么传输协议。UDP, TCP, SCTP, 等等。对于ZeroMQ，你通常只想要TCP。

* 填写iana.org上的申请，提供所有必要的信息。

* 然后IANA将通过电子邮件继续这一过程，直到你的申请被接受或拒绝。

请注意，你并不要求一个特定的端口号，IANA会给你分配一个。因此，明智的做法是在你运送软件之前而不是之后开始这个过程。
