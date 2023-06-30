# stark证明实操python实现02

# 第二部分: 约束

-   [视频教程 (youtube)](https://www.youtube.com/watch?v=fg3mFPXEYQY "视频教程 (youtube)")
-   [幻灯片 (PDF)](https://starkware.co/wp-content/uploads/2021/12/STARK101-Part2.pdf "幻灯片 (PDF)")

在这一部分中，我们将在轨迹`a` 上创建一组约束。
当且仅当轨迹表示FibonacciSq的有效计算时，约束将是轨迹单元格中的多项式（而不是有理函数）表达式。

我们将分三步实现：1.首先指定我们关心的约束（**FibonacciSq约束**）。2. 将FibonacciSq约束转换为**多项式约束**。3. 将它们转换为表示多项式的**有理函数**当且仅当原始约束成立时。

## 步骤 1 - FibonacciSq 约束

对于 `a` 是 FibonacciSq 序列的正确轨迹，可以证明我们的主张:

1.  第一个元素必须是1, $a[0]=1$。
2.  最后一个元素必须是 2338775057, $a[1022]=2338775057$。
3.  FibonacciSq规则必须适用,  对于每个 $i$ <1021, $a[i+2]=a[i+1]^2+a[i]^2$。

其计算结果恰好超过 G1，其中 G 是由 g 生成的“小”组。

## 步骤 2 - 多项式约束

回顾一下 `f` 是定义域轨迹的多项式， 其评估恰好在 G\\{ $g^{1023}$ } 上，其中 G={ $g^i:0≤i≤1023$ } 是个 "小的" 群，通过生成元g获取。&#x20;

&#x20;我们现在以多项式约束的形式重写上述三个约束`f`:

1.  $a[0]=1$ 可转换为多项式 $f(x)−1$, 当 $x=g^0$ (注意 $g^0$ 为 1)评估为0。
2.  $a[1022]=2338775057$ 可转换为多项式 $f(x)−2338775057$, 当 $x=g^{1022}$ 评估为 0。&#x20;
3.  $a[i+2]=a[i+1]^2+a[i]^2$ 对于每个 $i<1021$ 可转换为多项式 $f(g^2⋅x)−(f(g⋅x))^2−(f(x))^2$, 对于 $x∈G$\\ $\{g^{1021},g^{1022},g^{1023}\}$ ，评估为 0 。

### 开始动手&#x20;

首先，由于这是与第 1 部分不同的笔记，让我们运行以下代码，使此处的所有变量都具有正确的值。请注意，它最多可能需要 30 秒，因为它会重新运行多项式插值。

```纯文本
from channel import Channel
from field import FieldElement
from merkle import MerkleTree
from polynomial import interpolate_poly, X, prod
from tutorial_sessions import part1

a, g, G, h, H, eval_domain, f, f_eval, f_merkle, channel = part1()
print('Success!')
```

你将获得三个约束中的每一个作为两个多项式的商，确保余数是零的多项式。

## 步骤3 - 有理函数 (实际的多项式)

上面的每个约束都由一个多项式 $u(x)$ 表示，该多项式 $u(x)$ 假定在群 G 的某些元素上评估为 0。也就是说，对于某些 $x_0,…,x_k∈G$ , 我们声称

$u(x_0)=…=u(x_k)=0$

(请注意，对于前两个约束, $k=0$ 因为他们只提到一点，而第三个约束 $k=1021$)。

这相当于说 $u(x)$ 是可除的， 作为一个多项式, 由所有 $\{{(x−x_i)}\}_{i=0}^k$ 组成，或者有等效于下公式

$$
\prod_{i=0}^k(x-x_i)
$$

因此，上述三个约束中的每一个都可以写成该形式的有理函数：

$$
\frac{u(x)}{\prod_{i=0}^k(x-x_i)}
$$

对于相应 $u(x)$ 和 $\{{x_i}\}_{i=0}^k$。在这一步中，我们将构造这三个有理函数并证明它们确实是多项式。

## 第一个约束The First Constraint:

在第一个约束中， $f(x)-1$ 并且 $\{x_i\}=\{1\}$

我们现在将构建**多项式** $p_0(x)=\frac{f(x)-1}{x-1}$ ，确保 $f(x)-1$ 确实可以被 $(x-1)$ 整除。

```纯文本
numer0 = f - 1
denom0 = X - 1
```

事实上 $f(x)-1$ 有一个根为1在意味着它可被 $(x-1)$ 整除。运行以下代码块以说服自己 `numer0` 模 `denom0` 的余数为0，因此除法确实会产生一个多项式：

```纯文本
numer0 % denom0

```

运行以下代码块以通过将 `numer0` 除以 `denom0` 来构造 `p0`，即表示第一个约束的多项式。

```纯文本
p0 = numer0 / denom0

```

跑测试：

```纯文本
assert p0(2718) == 2509888982
print('Success!')

```

## 第二个约束:

构建多项式 $p_1(x)=\frac{f(x)-2338775057}{x-g^{1022}}$ ，类似的

```纯文本
numer1 = f - 2338775057
denom1 = X - g**1022
p1 = numer1 / denom1

```

跑测试：

```纯文本
assert p1(5772) == 232961446
print('Success!')

```

## 第三个约束:

构建第三个约束的有理函数相对复杂一些

$$
p_2(x)=\frac{f(g^2\cdot x)-(f(g\cdot x))^2-f(x)^2}{\prod_{i=0}^{1020} (x-g^{i})}
$$

可以重写其分母，以便整个表达式更易于计算：

$$
\frac{f(g^2\cdot x)-(f(g\cdot x))^2-f(x)^2}{\frac {(x^{1024}-1)} {(x-g^{1021})(x-g^{1022})(x-g^{1023})}}
$$

这是从以下等式得出的

$$
{\prod_{i=0}^{1023} (x-g^{i})}=x^{1024}-1
$$

```纯文本
lst = [(X - g**i) for i in range(1024)]
prod(lst)

```

有关更多信息，请参阅博客文章 [Arithmetization II](https://medium.com/starkware/arithmetization-ii-403c3b3f4355 "Arithmetization II")。

让我们暂停一下，看一个关于多项式如何组成的简单例子。之后我们将生成第三个约束。

### 组合多项式 (a detour)

创建两个多项式 $q(x)=2x^2+1，r(x)=x-3$:

```纯文本
q = 2*X ** 2 + 1
r = X - 3

```

把q r组合产生一个新的多项式：

$$
q(r(x))=2(x-3)^2+1=2x^2-12x+19
$$

### 返回到多项式约束

以类似于构造 `p0` 和 `p1` 的方式构造第三个约束 `p2`。过程中，验证 $g^{1020}$ 是**分子**的根，而 $g^{1021}$ 不是。

```纯文本
numer2 = f(g**2 * X) - f(g * X)**2 - f**2
print("Numerator at g^1020 is", numer2(g**1020))
print("Numerator at g^1021 is", numer2(g**1021))
denom2 = (X**1024 - 1) / ((X - g**1021) * (X - g**1022) * (X - g**1023))

p2 = numer2 / denom2
```

跑测试:

```纯文本
assert p2.degree() == 1023, f'The degree of the third constraint is {p2.degree()} when it should be 1023.'
assert p2(31415) == 2090051528
print('Success!')
```

运行以下代码块观察约束多项式的次数，均小于 1024 。这在下一部分很重要。

```纯文本
print('deg p0 =', p0.degree())
print('deg p1 =', p1.degree())
print('deg p2 =', p2.degree())
```

## 步骤4 - 组合多项式

回顾一下，我们正在将检查三个多项式约束的有效性问题转换为检查每个有理函数 $p_0,p_1,p_2$ 是多项式。

我们的协议使用称为 `FRI` 的算法来执行此操作，这将在下一部分中讨论。

为了使证明简洁（简短），我们更愿意只使用一个有理函数而不是三个。 为此，我们采用以下的随机线性组合 $p_0,p_1,p_2$ 称为**组合多项式**（简称CP）：

$$
CP(x)=\alpha_0\cdot p_0(x)+\alpha_1 \cdot p_1(x)+\alpha_2 \cdot p_2(x)
$$

$ \alpha_0,\alpha_1,\alpha_2  $ 是从验证者获得的随机字段元素，在我们的例子中是从信道(channel)获得的。

证明（有理函数） $CP$ 是一个多项式，也就高概率证明每个 $p_0,p_1,p_2$

本身也是对的多项式。

在下一部分中，你将为等效事实生成证明。 但让我们先使用`Channel.receive_random_field_element `创建 CP 以获得$\alpha_i$:

```纯文本
def get_CP(channel):
    alpha0 = channel.receive_random_field_element()
    alpha1 = channel.receive_random_field_element()
    alpha2 = channel.receive_random_field_element()
    return alpha0*p0 + alpha1*p1 + alpha2*p2
```

跑测试：

```纯文本
test_channel = Channel()
CP_test = get_CP(test_channel)
assert CP_test.degree() == 1023, f'The degree of cp is {CP_test.degree()} when it should be 1023.'
assert CP_test(2439804) == 838767343, f'cp(2439804) = {CP_test(2439804)}, when it should be 838767343'
print('Success!')
```

### 提交组合多项式

最后，我们评估 $cp$ 在评估定义域 (`eval_domain`) 上，在其上构建一棵 Merkle 树，并通过信道发送其根。 这类似于我们在第 1 部分末尾所做的 LDE 轨迹上的提交。

```纯文本
def CP_eval(channel):
    CP = get_CP(channel)
    return [CP(d) for d in eval_domain]
```

在评估上构造一个默克尔树，并通过信道发送其根。

```纯文本
channel = Channel()
CP_merkle = MerkleTree(CP_eval(channel))
channel.send(CP_merkle.root)
```

测试代码：

```纯文本
assert CP_merkle.root == 'a8c87ef9764af3fa005a1a2cf3ec8db50e754ccb655be7597ead15ed4a9110f1', 'Merkle tree root is wrong.'
print('Success!')
```
