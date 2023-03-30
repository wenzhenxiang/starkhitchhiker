# stark证明实操python实现01

本教程基于stark101教程翻译，可参考以下链接

[https://github.com/starkware-industries/stark101](https://github.com/starkware-industries/stark101 "https://github.com/starkware-industries/stark101")

### [在线运行](https://hub.ovh2.mybinder.org/user/starkware-industries-stark101-25kx5my7/lab/tree/tutorial/NotebookTutorial.ipynb "在线运行")

我们要用STAEK 证明有限域内的FibonacciSq 序列，这个FibonacciSq序列定义的关系为 $a_{n+2}=a_{n+1}^2+a_n^2$

我们的代码将生成STARK证明，证明以下陈述(statement)：

我知道一个域元素 $X \isin \mathbb F$ , 从1开始的FibonacciSq序列，第1023个元素 $X$ 是2338775057。

### 基础

### FieldElement 类（class）

我们用 `FieldElement` 类代表域元素，你可以从整型构造`FieldElement` 的实例，然后进行加减乘除，求逆等操作，这个域大小为 $\mathbb F_{3221225473}$ $(3221225473=3*2^{30}+1)$ ，所有操作都是基于模3221225473运算的。

```纯文本
from field import FieldElement
FieldElement(3221225472) + FieldElement(10)
```

## FibonacciSq 轨迹（trace）

首先，让我们构造一个长度为 1023 的列表 `a`，其前两个元素将分别是表示 1 和 3141592 的 FieldElement 对象。接下来的 1021 个元素由前两元素推导得出。`a`称为FibonacciSq的执行轨迹

```纯文本
a = [FieldElement(1), FieldElement(3141592)]
while len(a) < 1023:
    a.append(a[-2] * a[-2] + a[-1] * a[-1])
```

### 测试你的代码

运行下面的代码块测试填充到 `a` 的元素是否正确，注意这其实是个验证器，尽管非常幼稚与不简洁，因为它是逐个检查元素，从而确保是正确的。

```纯文本
assert len(a) == 1023, 'The trace must consist of exactly 1023 elements.'
assert a[0] == FieldElement(1), 'The first element in the trace must be the unit element.'
for i in range(2, 1023):
    assert a[i] == a[i - 1] * a[i - 1] + a[i - 2] * a[i - 2], f'The FibonacciSq recursion rule does not apply for index {i}'
assert a[1022] == FieldElement(2338775057), 'Wrong last element!'
print('Success!')

```

我们现在想把该序列看作是对次数(degree)为1022(由于Unisolvence 定理)的多项式 $f$ 的评估。我们将选择定义域(domain)作为某个子群 $G \subseteq \mathbb F ^×$ ，大小为 1024。

（回想一下, $\mathbb F^×$ 表示为 $\mathbb F$ 的乘法群，我们从中得到，它包含大小为 1024 的子群，因为 $\mathbb F^×$ 是大小为
$3*2^{30}$ 的循环群，因此它包含一个大小 $2^i$ 的子群，对于任何 $0≤i≤30$ )。

### 找一d个大小为1024的群

如果我们找到一个元素 $g∈\mathbb F$ 他的 (乘法) 阶为1024, 那么 $ g  $ 可以生成一个群，该 `FieldElement` 类提供了静态方法 `generator()` 返回一个元素作为的 $\mathbb F^×$ 生成元（它的阶为$|\mathbb F^×|$）。

1.  使用它获取 $G$ 的生成元  $g$ &#x20;
2.  创建一个为 `G` 的列表包含所有 $G$ 的元素, 例如 $G[i]:=g^i$

提示: 当 $k$ 除 $|\mathbb F^×|$ , $g^k$ 生成一个大小为 $\frac {|\mathbb F^×|} {k}$ 的群。

```纯文本
g = FieldElement.generator() ** (3 * 2 ** 20)
G = [g ** i for i in range(1024)]
```

运行下一个代码块测试你的代码

```纯文本
# Checks that g and G are correct.
assert g.is_order(1024), 'The generator g is of wrong order.'
b = FieldElement(1)
for i in range(1023):
    assert b == G[i], 'The i-th place in G is not equal to the i-th power of g.'
    b = b * g
    assert b != FieldElement(1), f'g is of order {i + 1}'
    
if b * g == FieldElement(1):
    print('Success!')
else:
    print('g is of order > 1024')
```

### 多项式类

我们提供一个类叫 `Polynomial`。 最简单的构建一个 `Polynomial` 使用变量 `X` (注意是大写的 `X`) 代表的正常变量 $x$ :

```纯文本
from polynomial import X
# The polynomial 2x^2 + 1.
p = 2*X**2 + 1
# Evaluate p at 2:
print(p(2))
# Type a polynomial's name, on its own, in the last line of a cell to display it
p
```

### 多项式插值(Interpolating a Polynomial)

我们 `polynomial` 模块提供了拉格朗日插值功能，它的参数包括:

-   x\_values: G的x值，他们的多项式值是已知的。\[列表]
-   y\_values: 对应的y值。\[列表]

他返回唯一的次数(degree)小于 `len(x_values)` 对所有i 的`x_values[i]` 值求值为`y_values[i]`的多项式实例。

运行以下代码块以获取有关函数 `interpolate_poly` 的帮助。

```纯文本
from polynomial import interpolate_poly
interpolate_poly?
```

假设`a` 包含一些关于`G` （除了`G[-1]`以外，因为`a`少一个元素）多项式的值，使用 `interpolate_poly()` 得到`f`然后获取它在`FieldElement(2)`的值。

```纯文本
from polynomial import interpolate_poly

f = interpolate_poly(G[:-1], a)
v = f(2)

```

跑测试:

```纯文本
assert v == FieldElement(1302089273)
print('Success!')
```

## 在更大的定义域上进行评估

轨迹(trace)，被视为多项式 $f$ 在 $G$ 上的评估, 可以在更大的定义域上评估 $f$ 扩展，从而创建Reed-Solomon 纠错码。

### 陪集（Cosets）

为此，我们必须决定一个更大的定义域来评估 $f$ 。我们将在8倍G大小的定义域上工作。

这样一个定义域的自然选择是采取一些大小为8192的群 $H$ (存在是因为8192能除 $|\mathbb F^×|$ )，并通过 $\mathbb F^×$ 的生成元对其进行移位 , 从而获得 $H$ 的 [陪集coset](https://en.wikipedia.org/wiki/Coset "陪集coset") 。

创建一个列表 `H` 包括所有 $H$ 的元素，并将它们中的每一个乘以 $\mathbb F^×$ 的生成元，得到一个名为 `eval_domain` 的列表。 换句话说，eval\_domain = $\{w⋅h^i|0≤i<8192\}$ , $h$ 是 $H$ 的生成元， $w$ 是 $\mathbb F^×$ 的生成元。

提示: 你已经知道如何获取H - 用前面获取 $G$ 同样的方式。

```纯文本
w = FieldElement.generator()
h = w ** ((2 ** 30 * 3) // 8192)
H = [h ** i for i in range(8192)]
eval_domain = [w * x for x in H]
```

跑测试：

```纯文本
from hashlib import sha256
assert len(set(eval_domain)) == len(eval_domain)
w = FieldElement.generator()
w_inv = w.inverse()
assert '55fe9505f35b6d77660537f6541d441ec1bd919d03901210384c6aa1da2682ce' == sha256(str(H[1]).encode()).hexdigest(),\
    'H list is incorrect. H[1] should be h (i.e., the generator of H).'
for i in range(8192):
    assert ((w_inv * eval_domain[1]) ** i) * w == eval_domain[i]
print('Success!')
```

### 在陪集上评估

是时候使用 `interpolate_poly` 和 `Polynomial.poly` 来评估陪集。请注意，它是在我们的 Python 模块中原生实现的，因此插值可能需要长达一分钟的时间。
事实上，插值和评估多项式轨迹是 STARK 协议中计算量最大的步骤之一，即使使用更有效的方法（例如 FFT，傅里叶变化）也是如此。

```纯文本
f = interpolate_poly(G[:-1], a)
f_eval = [f(d) for d in eval_domain]
```

## 承诺

我们将使用基于Sha256的Merkle树作为我们的承诺方式。你可以在`MerkleTree`类中获得它的简单实现。运行下一个代码块（为了本教程的目的，这也用于测试到目前为止整个计算的正确性）：

```纯文本
from merkle import MerkleTree
f_merkle = MerkleTree(f_eval)
assert f_merkle.root == '6c266a104eeaceae93c14ad799ce595ec8c2764359d7ad1b4b7c57a4da52be04'
print('Success!')
```

## 信道(Channel)

从理论上讲，STARK 证明系统是双方之间交互的协议 - 证明者和验证者。在实践中，我们使用 Fiat-Shamir Heuristic 式将这种交互式协议转换为非交互式证明。在本教程中，你将使用 `Channel` 类来实现此转换。在证明者（你正在编写的）将发送数据的意义上来说，此信道取代了验证者，并接收随机数或随机 **FieldElement** 实例。

这段简单的代码实例化一个信道对象，将默克尔树的根发送到它。稍后，可以调用信道对象以提供随机数或随机域元素。

```纯文本
from channel import Channel
channel = Channel()
channel.send(f_merkle.root)
```

最后 - 你可以通过打印成员`Channel.proof`来检索到目前为止的证明（即，在信道中传递到某个点的所有内容）

```纯文本
print(channel.proof)
```
