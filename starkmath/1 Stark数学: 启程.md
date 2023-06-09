# Stark数学：启程

原文链接：

[https://medium.com/starkware/stark-math-the-journey-begins-51bd2b063c71](https://medium.com/starkware/stark-math-the-journey-begins-51bd2b063c71 "https://medium.com/starkware/stark-math-the-journey-begins-51bd2b063c71")

## 第一部分

StarkWare 的使命是将 STARK 带入现实世界。这是解释 STARKs 背后的理论及其实现的系列文章中的第一篇。我们从头开始，并将在后续帖子中增加相关数学知识。

![](https://miro.medium.com/max/1400/1*1aLu20aptPeUTDg_E8KtmA.jpeg)

乔安娜·科辛斯卡( [Joanna Kosinska)](https://unsplash.com/photos/1_CMoFsPfso?utm_source=unsplash\&utm_medium=referral\&utm_content=creditCopyText "Joanna Kosinska)")在[Unsplash上拍摄的照片](https://unsplash.com/collections/4189288/strak-math-1-journey-begins?utm_source=unsplash\&utm_medium=referral\&utm_content=creditCopyText "Unsplash上拍摄的照片")

# 一、介绍

计算完整性 (CI，Computational Integrity) 是商业的基本属性。简单来说，就是某个计算的输出是正确的。CI 使我们能够信任呈现给我们的账户余额或商店的账单。这篇文章将深入探讨公链如何在去信任的情况下实现 CI，公链们在可扩展性和隐私方面为此付出的巨大代价，以及 STARKs 如何挽救局面。

# 二、信任与验证

## “旧世界”：信任或委托问责

金融系统（银行、经纪人、交易所等）需要诚信运作，以服务于其社会职能。什么机制激励他们诚信经营？“旧世界” *假设信任* 视为诚信的代表。我们相信银行、养老基金等会诚实经营。让我们深入探讨信任的基础，忽略“表面诚信” —— 高大的建筑和华丽的西装 —— 会给我们留下深刻印象。从纯粹理性和功利主义的角度来看，阻止金融体系没收我们所有资金，将被认为是社会耻辱、面临监禁和罚款的威胁。

还有 —— 声誉，它可以吸引未来的客户并产生未来的利润。通过在财务报表上签名，“旧世界”中的人们将他们的个人自由、现有和未来的财务作为诚信的*抵押品*，而我们公众则基于这种安排建立信任。这种完整性的验证**委托**给会计师、审计师和监管机构等专家。我们将其称为委派问责。这不是一个糟糕的系统：它忠实地为现代经济服务了相当长的一段时间。

“旧世界”方法的一个新变体是可信执行环境 (TEE)。受信任的硬件制造商（如英特尔）生产的物理机器（如 SGX 芯片）不能偏离指定的计算，并使用只有该物理机器知道的密钥签署正确的状态。完整性现在基于对硬件及其制造商的信任，并假设不可能从此类物理设备中提取密钥。

## “新世界”：验证或包容性问责

区块链提供了一种更直接的方式来实现完整性，这被“不信任，验证”的座右铭所捕捉。这个“新世界”不需要表面诚信，它不依赖会计师，它的开发人员和网络维护人员也不用抵押个人自由来获得公众信任。完整性由包容性问责制保证：具有标准计算设置的节点（连接网络的笔记本电脑）应该能够验证系统中所有交易的完整性。

在公链中验证 CI 的流行方法是通过简单重放：要求所有节点重新执行（重放）验证每笔交易的计算。这种简单形式的包容性问责制导致了两个直接挑战：

-   **隐私：** 如果每个人都可以检查所有交易，那么隐私可能会受到损害。缺乏隐私会阻碍企业，因为这意味着敏感信息可能不会保密。它也会阻碍个人，因为它侵蚀了[人的尊严](https://en.wikipedia.org/wiki/The_Right_to_Privacy_\(article\) "人的尊严")。
-   **可扩展性：** 要求系统向标准笔记本电脑负责意味着它不能通过仅将其移动到更大的计算机和更大的带宽来扩展。这导致系统吞吐量的严重限制。

证明系统（接下来讨论）是解决这两个挑战的绝佳解决方案。零知识 (ZK) 证明系统现在是解决区块链隐私问题的成熟工具，并在 Zcash \[ [1](https://z.cash/blog/shielded-ecosystem/ "1") , [2](https://z.cash/technology/ "2") , [3](https://z.cash/technology/zksnarks/ "3") ] 的几篇文章中得到了很好的解释。尽管 ZK-STARK 提供零知识，但本文不会讨论它，而是关注可扩展性和透明度（STARK 中的 S 和 T）。

# 三、证明系统

证明系统始于 1985 年 Goldwasser、Micali 和 Rackoff 引入的交互式[证明](https://en.wikipedia.org/wiki/Interactive_proof_system "证明")(IP) 模型。交互式证明是涉及两种实体的协议：*证明者和验证者*，它们通过发送数轮进行交互消息。证明者和验证者的目标是相互冲突的：证明者希望让验证者相信某项计算的完整性，而验证者是受公众委托承担辨别真假任务的可疑看门人。证明者和验证者交互通信，轮流相互发送消息。这些消息取决于被证明的陈述、先前的消息，并且还可能使用一些随机数。在证明者方面，随机性用于实现零知识，而在验证者方面，需要随机性来生成对证明者的查询。在交互过程结束时，验证者输出一个决定，要么接受新状态，要么拒绝它。

一个很好的类比是当一方提出索赔而其对手方质疑其有效性时，法院实行的审查过程。为了使索赔被接受，索赔人（证明人）对审查员（验证人）询问的回答必须一致且有效。审查过程应该揭示任何陈述与现实之间的不匹配，从而将其揭示为虚假。

如果在将系统从状态A更新到状态B时，我们说*证明*系统解决了 CI ，则以下属性成立：

-   **完整性：** 如果证明者确实知道如何以有效的方式将状态从A更改为B，那么证明者将设法说服验证者接受更改。
-   **健全性：** 如果证明者不知道如何将状态从A更改为B，那么验证者将注意到交互中的不一致并拒绝建议的状态转换。仍然存在很小的误报概率，即验证者接受无效证明的概率。该概率是一个系统安全参数，可以设置为可接受的水平，例如1/(2¹²⁸)，类似于连续五次赢得强力球的几率。

这对属性对前面讨论的包容性问责原则具有重要意义。验证者可以接受证明者建议的状态转换，而不做任何关于证明者诚信的假设。事实上，证明者可以在有故障的硬件上运行，它可以是闭源的，并且可以在恶意实体控制的计算机上执行。唯一重要的<sup>1</sup>是证明者发送的消息导致验证者接受声明。如果是这样，我们就知道计算完整性是成立的。

# 四、STARK

到目前为止，已经有相当多的证明系统的理论构造以及实现。有些部署在加密货币中，例如[Zerocash](https://z.cash/technology/zksnarks/ "Zerocash") / [Zcash](https://z.cash/ "Zcash")使用的[SNARK](http://zerocash-project.org/paper "SNARK")，以及[Monero中部署的](https://ww.getmonero.org/ "Monero中部署的")[Bulletproofs](https://eprint.iacr.org/2017/1066 "Bulletproofs") (BP) 。（有关证明系统的一般信息，请访问[此处](https://zkp.science/ "此处")。）[STARK](https://eprint.iacr.org/2018/046 "STARK")的区别在于以下三个属性的组合：可扩展性（STARK 中的 S）、透明性（STARK 中的 T）和精益密码学(lean cryptography)。

## 可扩展性——验证的指数加速

可扩展性意味着两个效率属性同时保持：

-   **可扩展的证明者：** 证明者的运行时间是“接近线性的”，因为可信计算机只需重新执行计算并检查结果是否与某人声称的相符即可检查 CI。“开销”（生成证明所需的时间/运行计算所需的时间）的比率仍然相当低。
-   **可扩展验证器：** 验证器的运行时间是原生重放时间的对数多项式。换句话说，验证者的运行时间比简单地重放计算要小得多（回想一下，“重放”是当前实现包容性问责制的区块链方法）。

![](https://miro.medium.com/max/1400/1*UTKB4bILBsDzDMEGU9feYg.png)

将这种可扩展性概念应用于区块链。想象一下当区块链通过使用证明系统进行验证时，情况会如何，而不是当前通过简单重放进行验证的模式。证明者节点需要生成证明，而不是简单地发送要添加到区块链的交易，但是由于可扩展证明者，它的运行时间几乎与天真的重播解决方案的运行时间呈线性关系。Scalable Verifier 将受益于其验证时间的指数减少。此外，随着区块链吞吐量的扩大，大部分影响将由证明者节点（可以在专用硬件上运行，如矿工）承担，而构成网络中大部分节点的验证者几乎不会受到影响。

让我们考虑一个具体的假设示例，假设验证者时间（以毫秒为单位）的比例类似于交易数量 (tx) 的对数的平方。假设我们从10,000 tx/块开始。那么验证者的运行时间为

VTime *= (* log₂ *10,000)² \~ (13.2)² \~ 177 ms*

现在将块大小增加一百倍（达到1,000,000 tx/块）。验证者的新运行时间为

VTime *= (* log₂ *1,000,000)² \~ 20² \~ 400 ms*

换句话说，交易吞吐量增加**100倍**只会导致验证者的运行时间增加**2.25倍！**

在某些情况下，验证者仍需要下载并验证数据的可用性，这是一个线性时间过程，但下载数据通常比检查其有效性更便宜、更快。

## 透明度：无需对任何信任，对所有人的诚信

透明度意味着²没有可信的设置——在系统设置中没有使用秘密参数。透明度提供了许多好处。它消除了构成单点故障的参数设置生成过程。由于缺乏可信的设置，即使是控制“旧世界”金融体系的强大实体——大公司、垄断企业和政府——也可以证明 CI 并让公众接受他们的主张，因为没有已知的方法可以伪造 STARK 的虚假证明，即使是最强大的实体。在更具战术性的层面上，它使部署新工具和基础设施以及更改现有工具和基础设施变得更加容易，而无需复杂的参数生成仪式。最重要的是，[套用亚伯拉罕·林肯](https://en.wikipedia.org/wiki/Abraham_Lincoln's_second_inaugural_address "套用亚伯拉罕·林肯")的话说，透明的系统允许在不信任任何人的情况下运作，对所有人都保持诚信。

## 精益密码学：安全快速

STARK 在其安全性基础上具有最小的加密假设：安全加密和[抗冲突哈希函数](https://en.wikipedia.org/wiki/Collision_resistance "抗冲突哈希函数")的存在。许多这些原语今天作为硬件指令存在，精益密码学带来了另外两个好处：

-   **后量子安全：** STARK 似乎可以抵御高效的量子计算机。
-   **具体效率：** 对于给定的计算，STARK 证明者至少比 SNARK 和 Bulletproofs 证明者快10倍。*STARK 验证器至少比 SNARK 验证器快2倍，比 Bulletproof 验证器快10倍以上。随着 StarkWare 继续优化 STARK，这些比率可能会提高。然而，一个 STARK 证明的长度比相应的 SNARK 大100倍，比 BulletProofs 大20倍。*

# 五、总结

我们首先解释了公链的“新世界”，其中的座右铭是“不信任，验证”。包容性问责原则要求任何人都可以轻松验证金融系统的完整性，而不是“旧世界”的委托问责制。为了允许区块链扩展，我们需要能够验证计算完整性的方法，这些方法比简单重放更快。

STARKs 是一种特殊的证明系统，它提供可扩展性和透明度，并且基于精益密码学。它们的可扩展性和透明度允许对 CI 进行廉价且去信任的验证。这是 STARKs 的承诺，也是 StarkWare 的使命。

我们的下一篇文章将更深入地探讨构建 STARK 的数学。

感谢 Vitalik Buterin 和 Arthur Breitman 审阅了这篇博文的草稿。

*Michael Riabzev & Eli Ben-Sasson*

*StarkWare Industries*

翻译：Xiang

¹隐私保护 (ZK) 确实对证明者代码提出了要求，以确保证明者消息不会通过旁道泄露有关见证人的信息。但是稳健性不需要信任假设。

²形式上，透明的证明系统是所有验证者消息都是公共随机字符串的系统。这样的系统也被称为[Arthur-Merlin 协议](https://en.wikipedia.org/wiki/Arthur–Merlin_protocol "Arthur-Merlin 协议")。

³这种最低限度的加密假设适用于交互式 STARK (iSTARK)。非交互式 STARKs (nSTARKs) 需要 Fiat-Shamir 启发式算法，这是一种不同的野兽。
