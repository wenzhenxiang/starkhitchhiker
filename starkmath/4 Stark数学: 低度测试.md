# Stark数学:   低度测试

### 简洁的秘诀

原文链接:

[https://medium.com/starkware/low-degree-testing-f7614f5172db](https://medium.com/starkware/low-degree-testing-f7614f5172db "https://medium.com/starkware/low-degree-testing-f7614f5172db")

![](image/image_8hHYGZHFO7.png)

这是我们 STARK 数学系列的第四篇文章。如果你还没有读完之前文章，我们建议您在阅读这篇文章之前阅读帖子 1（旅途开始）、2（[*算术 I*](https://medium.com/starkware/arithmetization-i-15c046390862 "算术 I")[）](https://medium.com/starkware/arithmetization-i-15c046390862 "）")和 3（[*算术 II*](https://medium.com/starkware/arithmetization-ii-403c3b3f4355 "算术 II")）。到目前为止，我们已经解释了在 STARK 中，算术化过程如何使我们能够将计算完整性问题简化为低度测试问题。

低度测试指通过仅对函数进行少量查询来确定给定函数是否是某个有界度数的多项式的问题。低度测试已经研究了二十多年，是概率证明理论的核心工具。这篇文章的目的是更详细地解释低度测试，并描述FRI，我们在 STARK 中用于低度测试的协议。这篇文章假设你熟悉有限域上的多项式。

为了使这篇非常数学的帖子更容易消化，我们将某些段落标记如下：

> 包含对理解大局不必要的证明或解释的段落将被标记为这样

# 低度测试

在讨论低度测试之前，我们首先提出一个稍微简单的问题作为热身：给定一个函数，并要求我们通过查询在“少量”位置发挥作用。形式上，给定域 F 的子域 L 和度界限 d，我们希望确定一个函数 f:L➝F 是否等于度数小于 d 的多项式p(x)，即是否存在多项式

![](https://miro.medium.com/max/660/1*0XHZfLP9jd6AcQtzKGruHw.png)

在 F 上，对于 L 中的每个 a，p(a) = f(a)。对于具体值，你可能会想到一个非常大的字段，例如 2¹²⁸，而 L 的大小约为 10,000,000。

解决这个问题需要在整个子域L 中查询 f，因为 f 可能与 L 中除了某个位置之外的任何地方的多项式均一致。即使我们允许一个恒定的错误概率，查询的数量在 L 的大小上仍然是线性的。

因此，低度测试的问题实际上是对上述问题的一种近似放松，它足以构建概率证明，也可以通过在 |L| 中对数级别的查询数来解决。（请注意，如果 L≈10,000,000，则 log₂(L)≈23）。更详细地说，我们希望区分以下两种情况。

-   **函数f等于一个低度多项式。** 即，在 F 上存在一个度数小于 d 的多项式 p(x)，它在 L 上处处与 f 一致。
-   **函数 f 与所有低度多项式相去甚远。** 例如，我们需要修改至少10% 的 f 值，然后才能获得与度数小于 d 的多项式一致的函数。

请注意，还有另一种可能性——函数 f 可能略微接近低度多项式，但不等于 1。例如，5% 的值与低度多项式不同的函数不属于上述两种情况中的任何一种。然而，先前的算术化步骤（在我们之前的文章中讨论过）确保第三种情况永远不会出现。更详细地说，算术化表明，处理真实statement的诚实证明者将在第一种情况下出现，而试图“证明”虚假声明的（可能是恶意的）证明者很可能会出现在第二种情况下。

为了区分这两种情况，我们将使用概率多项式时间测试，在少量位置查询 f（我们稍后再讨论“少”的含义）。

> 如果 f 确实是低度，那么测试应该以1的概率接受。如果 f 远非低度，那么测试应该以高概率（接近1的概率）拒绝。更一般地，我们寻求保证，如果 f 与任何度数小于 d 的函数相差 δ（即须修改至少 δ|L| 个位置以获得度数小于 d 的多项式），则测试拒绝概率至少为 Ω(δ)。直观地说，δ 越接近于零，就越难区分这两种情况。

在接下来的几节中，我们描述了一个简单的测试，然后解释了为什么它在我们的设置中不够用，最后我们描述了一个更复杂的测试，它的效率呈指数级增长。后一种测试是我们在 STARK 中使用的测试。

# 直接测试

我们考虑一个简单的测试：它使用 d+1 次查询来测试一个函数是否（接近）一个度数小于 d 的多项式。该测试依赖多项式的性质：*任何度数小于 d 的多项式完全由其在域 F 的任何 d 个不同位置处的点确定*。这个性质是一个k度的多项式在 F中最多可以有k个根这一事实的直接结果。重要的是，查询的数量，即 d+1，可以显著的小于 f 的域的大小，即|L|。

我们首先讨论两个简单的特殊情况，以直观地了解测试在一般情况下的工作方式。

-   **常数函数的情况（d=1）。** 这对应于区分 f 是常数函数的情况（对于 F 中的某些 c，f(x)=c）和 f 远离任何常数函数的情况的问题。在这种特殊情况下，此时只需要2次查询测试：检查在固定位置$ z_1  $和随机位置 $w$，然后检查 $f(z_1)=f(w)$就可。简单来说，$ f(z_1)  $确定 f 的常数值，$ f(w)  $测试是否所有的 f 都接近这个常数值。
-   **线性函数的情况（d=2）。** 这对应于区分 f 是线性函数的情况（对于 F 中的某些 a,b，$f(x)=ax+b$）和 f 远离任何线性函数的情况的问题。在这种特殊情况下，有一个自然的 3 查询测试可能有效：在两个固定位置 $z_1,z_2$ 和随机位置 $ w  $查询 f，然后检查 $ (z_1,f(z_1)), (z_2,f (z_2)), (w,f(w))  $是共线的，即我们可以通过这些点画一条线。直观地说，$f(z_1)$ 和$  f(z_2) $ 的值决定了线，f(w) 测试是否所有的 f 都接近这条线。

上述特殊情况建议对**度数d 的一般情况进行**测试。在 d 个固定位置 $z_1,z_2,…,z_d$以及随机位置 $w$处查询 $f$ 。$z_0,z_1,…,z_d$处的 $ f  $值定义了一个在 F 上的度数小于 d 的唯一多项式 $h(x)$，它在这些点与 f 一致。然后测试检查 $h(w)=f(w)$。我们称之为***直接测试***。

根据定义，如果 f(x) 等于度数小于 d 的多项式 p(x)，则 h(x) 将与 p(x) 相同，因此直接测试以概率 1 通过。此属性称为“完美完备性”，这意味着这个测试只有一侧的错误。

我们只能争论如果 f 相差 δ - 远离与任何度数小于 d 的函数。（例如，考虑 δ=10%。）我们现在认为，在这种情况下，直接检验以至少 δ 的概率拒绝。事实上，让 𝞵 是随机选择w 的概率，即h(w)≠f(w) 。概率 𝞵 必须至少为 δ。

> 这是因为如果我们假设 𝞵 小于 δ ，那么我们推断 f 是 δ-接近 h，这与我们的假设相矛盾，即 f 是 δ - 远离任何度小于 d 的函数。

# 直接测试对我们来说是不够的

在我们的设置中，我们感兴趣的是测试编码计算轨迹的函数 f:L➝F，因此其度数 d（和域 L）非常大。仅仅运行直接测试，进行 d+1 查询，成本太高。为了获得 STARK 的指数级节省（验证时间与计算轨迹的大小相比），我们需要仅用 O(log d) 查询来解决这个问题，这比度数限制 d 呈指数级小。

不幸的是，这是不可能的，因为如果我们在少于 d+1 个位置查询 f，那么我们就无法得出任何结论。

> 看到这一点的一种方法是考虑函数 f:L➝F 的两种不同分布。在一个分布中，我们一致地选择一个精确度为 d 的多项式并在 L 上对其进行评估。在另一个分布中，我们一致地选择一个度数小于d的多项式并在 L 上对其进行评估。在这两种情况下，对于任何 d 位置$  z_1,z_2， …,z_d
>   $, 值$  f(z_1),f(z_2),…,f(z_d)  $是均匀且独立分布的。（我们把这个情况留给读者练习。）这意味着从信息上讲，我们无法区分这两种情况，即使需要进行测试（因为来自第一个分布的多项式应该被测试接受，而那些精确度为 d 的多项式与所有小于 d 的多项式相距甚远，因此应该都应该被拒绝）。

我们似乎有一个艰巨的挑战需要克服。

# 证明者来解决

我们已经看到，我们需要 d+1 次查询来测试函数 f:L➝F 是否接近于度数小于 d 的多项式，但是我们负担不起这么多查询。我们通过考虑稍微不同的设置来避免这种限制，这对我们来说就足够了。也就是说，当证明者可以提供关于函数 f 的有用辅助信息时，我们会考虑低度测试问题。我们将看到，在这种低度测试的“证明者辅助”设置中，我们可以实现查询数量的指数级改进，达到 O(log d)。

更详细地说，我们考虑在*证明者*和*验证者*之间进行的*协议*，其中（不受信任的）证明者试图说服验证者该函数是低度的。一方面，证明者知道*整个*函数 f 被测试。另一方面，验证者可以在少量位置查询函数 f，并且愿意从证明者那里获得帮助，但不相信证明者是诚实的。这意味着证明者可能会作弊而不遵守协议。但是，如果证明者确实作弊，则无论函数 f 是否为低度，验证者都有“拒绝”的自由。这里的重点是验证者不会相信 f 是低度的，除非这是真的。

请注意，上面描述的直接测试只是协议的特殊情况，其中证明者什么都不做，验证者在没有帮助的情况下测试功能。为了比直接测试做得更好，我们需要以某种有意义的方式利用证明者的帮助。

在整个协议中，证明者将希望使验证者能够在验证者选择的位置上查询辅助功能。这可以通过*承诺*(译者注：通过多项式解决这个问题，Starknet采用的是FRI)来实现，我们将在以后的博客文章中讨论这种机制。现在可以说证明者可以通过默克尔树提交其选择的函数，随后验证者可以请求证明者揭示已提交函数的任何位置集。这种承诺机制的主要特性是，一旦证明者提交了一个函数，它必须揭示正确的值并且不能作弊（例如，它不能在看到验证者的请求后决定函数的值是什么）。

# 将两个多项式情况下的查询次数减半

让我们从一个简单的示例开始，该示例说明证明者如何帮助将查询数量减少 2 倍。稍后我们将在此示例的基础上进行构建。假设我们有两个多项式 f 和 g，我们想测试它们的度数是否都小于 d。如果我们只是在 f 和 g 上单独运行直接测试，那么我们需要进行 2 \* (d + 1) 次查询。下面我们将描述如何在证明者的帮助下将查询数量减少到 (d + 1) 加上一个小阶项。

首先，验证者从字段中采样一个随机值𝛼，并将其发送给证明者。接下来，证明者通过承诺对多项式 $ h(x) = f(x) + ag(x)  $的域 L 的评估（回想一下 L 是函数 f 的域）进行评估（换句话说，证明者将计算并发送 Merkle 树的根，其叶子是 L 上 h 的值）。验证者现在通过直接测试测试h的度数小于 d，这需要 d+1 个查询。

直观地说，如果$f$或$g$的度数至少为 d，那么 h 很有可能也是如此。例如，考虑对于某些 n≥d，f 中 xⁿ 的系数不为零的情况。那么，至多有一个𝛼（由验证者发送），其中xⁿ在h中的系数为零，这意味着h的度数小于d的概率大约为1/|F|。如果场足够大（例如，|F|>2¹²⁸），错误的概率可以忽略不计。

然而，情况并非如此简单。原因是，正如我们所解释的，我们不能从字面上检查 h 是否是度数小于 d 的多项式。相反，我们只能检查 h 是否接近这样的多项式。这意味着上面的分析是不准确的。是否有可能 f 将远离低度多项式，而线性组合 h 将接近 1，在 𝛼 上具有不可忽略的概率？在温和的条件下，答案是否定的（这是我们想要的），但这超出了本文的范围；我们将感兴趣的读者推荐给[这篇论文](https://acmccs.github.io/papers/p2087-amesA.pdf "这篇论文")和[这篇论文](https://eccc.weizmann.ac.il/report/2017/134/ "这篇论文")。

此外，验证者如何知道证明者发送的多项式h的形式为 f(x)+𝛼 g(x)？一个恶意的证明者可能会通过发送一个多项式来作弊，该多项式确实是低度的，但与验证者要求的线性组合不同。如果我们已经知道 h 接近于一个低度多项式，那么测试这个低度多项式是否具有正确的形式很简单：验证者随机抽样 L 中的一个位置，查询f、*g、* h在 z 处，并检查方程 h(z)=f(z)+𝛼 g(z) 是否成立。该测试应重复多次以提高测试的准确性，但误差会随着我们制作的样本数量呈指数级缩小。因此，此步骤仅将查询的数量（到目前为止为 d+1）增加了一个较小的项。

# 将多项式拆分为两个较小度数的多项式

我们看到，在证明者的帮助下，我们可以通过少于 2 \*(d+1) 个查询来测试两个多项式的度数小于 d。我们现在描述如何将一个度数小于 d 的多项式变成两个度数小于 d/2 的多项式。

让 f(x) 是度数小于 d 的多项式，并假设 d 是偶数（在我们的设置中，这不会失去一般性）。对于度数小于 d/2 的两个多项式 g(x) 和 h(x)，我们可以写出 f(x)=g(x²)+xh(x²)。实际上，我们可以让 g(x) 是从 f(x) 的偶数系数获得的多项式，而 h(x) 是从 f(x) 的奇数系数获得的多项式。例如，如果 d=6 我们可以写

![](https://miro.medium.com/max/650/1*gpXFISzO37_14vos9zaeew.png)

意思就是

![](https://miro.medium.com/max/600/1*DLr8g2RoFB7aJefoScItaQ.png)

和

![](https://miro.medium.com/max/600/1*7FTEBsQuwKhKHa7vcmIanw.png)

这是一个用于多项式评估的 n\*log(n) 算法（改进了原生的 $n^2$ 算法）。

# FRI 协议

我们现在将上述两个想法（用一半的查询测试两个多项式，并将一个多项式分成两个较小的多项式）组合成一个协议，该协议仅使用 O(log d) 查询来测试函数 f 是否具有（更准确地说，接近为) 度小于 d 的函数。该协议被称为 FRI（代表**Fast** Reed — Solomon **I**ninteractive Oracle Proof of Proximity） **，** 感兴趣的读者可以[在这里](https://eccc.weizmann.ac.il/report/2017/134/ "在这里")阅读更多信息。为简单起见，下面我们假设 d 是 2 的幂。该协议由两个阶段组成：*提交阶段*和*查询阶段*。

## 提交阶段

证明者将原多项式f₀(x)=f(x)拆分成两个度数小于d/2的多项式g₀(x)和h₀(x)，满足f₀(x)=g₀(x²)+xh₀(x² ）。验证者采样一个随机值𝛼₀，将其发送给证明者，并要求证明者承诺多项式 f₁(x)=g₀(x) + 𝛼₀h₀(x)。请注意，f₁(x) 的度数小于 d/2。

我们可以通过将 f₁(x) 拆分为 g₁(x) 和 h₁(x) 来继续递归，然后选择一个值 𝛼₁，构造 f₂(x) 等等。每次，多项式的度数减半。因此，在 log(d) 步骤之后，我们得到了一个常数多项式，证明者可以简单地将常数值发送给验证者。

关于域的注释：要使上述协议起作用，我们需要对于域L 中的每个z，都认为−z 也在L 中。此外，f₁(x) 上的承诺不是在L内而是 L²={x²: x ∊ L}内。由于我们迭代地应用 FRI 步骤，L² 还必须满足 {z, -z} 属性，依此类推。这些自然代数要求很容易通过域 L 的自然选择来满足（例如，选择 2 为幂的乘法子群），实际上与我们无论如何都需要的那些一致，以便从有效的 FFT 算法中受益（使用STARK 中的其他地方，例如，对执行轨迹进行编码）。

## 查询阶段

我们现在必须检查证明者没有作弊。验证者对 L 中的一个随机 z 进行采样并查询 f₀(z) 和 f₀(-z)。这两个值足以确定 g₀(z²) 和 h₀(z²) 的值，从以下两个“变量”g₀(z²) 和 h₀(z²) 中的两个线性方程可以看出：

![](https://miro.medium.com/max/600/1*IWLIX1L6aNMQLh7SEd6SQg.png)

验证者可以求解这个方程组并推导出 g₀(z²) 和 h₀(z²) 的值。因此，它可以计算 f₁(z²) 的值，这是两者的线性组合。现在验证者查询 f₁(z²) 并确保它等于上面计算的值。这表明证明者在提交阶段发送的对 f₁(x) 的承诺确实是正确的。验证者可以继续，通过查询 f₁(-z²)（回想一下 (-z²)∊ L² 并且对 f₁(x) 的承诺是在 L² 上给出的）并从中推断出 $f_2(z^4)$。

验证器以这种方式继续，直到它使用所有这些查询最终推导出 $f_{log d}(z)$ 的值(译者注：原文此处由于采用的medium，不能写公式，中文这里有修正)。但是，请记住 $f_{log d}(z)$ 是一个常数多项式，其常数值由证明者在提交阶段（在选择 $z$ 之前）发送。验证者应检查证明者发送的值是否确实等于验证者从对先前函数的查询中计算出的值。

总体而言，查询次数仅在度数限制 d 中呈**对数关系。**

> 要了解证明者不能作弊的原因，请考虑在 {z,-z} 对上 90%  f₀ 为零的问题，即 f₀(z) = f₀(-z) = 0（调用这些“好”数对），其余 10%（“坏”的数对）非零。随机选择的 z 有 10% 的概率落在坏对中。请注意，只有一个𝛼会导致 f₁(z²)=0，其余的会导致 f₁(z²)≠0。如果证明者在 f₁(z²) 的值上作弊，它就会被抓住，所以我们假设不是这样。因此，很有可能 (f₁(z²), f₁(-z²)) 在下一层也将是坏对（f₁(-z²) 的值并不重要，因为 f₁(z²)≠0）。这一直持续到最后一层，该值很有可能不为零。

另一方面，由于我们从一个具有 90% 零的函数开始，证明者不太可能接近零多项式以外的低度多项式（我们不会在这里证明这一事实）。特别是，这意味着证明者必须发送 0 作为最后一层的值。但是，验证者有大约 10% 的概率抓住证明者。这只是一个非正式的论证，有兴趣的读者可以[在这里](https://eccc.weizmann.ac.il/report/2017/134/ "在这里")找到一个严格的证明。

在到目前为止描述的测试（以及上述分析）中，验证者捕获恶意证明者的概率仅为 10%。换句话说，错误概率是 90%。这可以通过对几个独立采样的 z 重复上述查询阶段来以指数方式改进。例如，通过选择 850 个 z，我们得到 $2^{-128}$ 的错误概率，实际上为零。

# 总结

在这篇文章中，我们描述了低度测试的问题。直接解决方案（测试）需要太多查询才能达到 STARK 要求的简洁性。为了达到对数查询复杂度，我们使用了一种称为 FRI的**交互式协议，证明者在该协议中添加了更多信息，以使验证者相信该函数确实是低度的。** 至关重要的是，FRI 使验证者能够通过在规定度数内成对数的多个查询（和轮次交互）来解决低度测试问题。下一篇文章将总结最后三篇文章，并解释我们如何去除 FRI 和协议早期部分中使用的交互方面，以获得简洁的非交互证明。

*Alessandro Chiesa & Lior Goldberg*

*StarkWare*

翻译：Xiang
