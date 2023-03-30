# stark证明实操python实现03

# 第三部分: FRI 承诺

-   视频教程[ (youtube)](https://www.youtube.com/watch?v=gd1NbKUOJwA " (youtube)")
-   幻灯片[ (PDF)](https://starkware.co/wp-content/uploads/2021/12/STARK101-Part3.pdf " (PDF)")

### 加载之前代码

运行下面代码块以加载相关变量。像往常一样 - 运行需要一段时间。

```纯文本
from channel import Channel
from field import FieldElement
from merkle import MerkleTree
from polynomial import interpolate_poly, Polynomial
from tutorial_sessions import part1, part2

cp, cp_eval, cp_merkle, channel, eval_domain = part2()
print("Success")
```

## FRI 折叠（Folding)

我们在这部分的目标是构建 FRI 层并在其上提交。要获得每一层，我们需要:

1.  为该层生成定义域（来自上一层的定义域）。
2.  为该层生成多项式（从前一层的多项式和定义域）。
3.  评估所述定义域上的所述多项式——**这是下一个 FRI 层**。

### 生成定义域（Domain）

第一个 FRI 定义域只是你在第 1 部分中生成的`eval_domain`，即一组 8192 阶的陪集(coset)。每个后续 FRI 的定义域都是通过获取前一个 FRI 定义域的前半部分（丢弃后半部分）获得的 , 并对其每个元素进行平方。

正式地 - 我们通过采取以下方式获得了`eval_domain`：

$$
w,w\cdot h,w\cdot h^2,...,w\cdot h^{8191}
$$

因此，下一层将是：

$$
w^2,(w\cdot h)^2,(w\cdot h^2)^2,...,(w\cdot h^{4095})^2
$$

请注意，取 `eval_domain` 中每个元素的后半部分的平方与取前半部分的平方产生的结果完全相同。 对于下一层也是如此。 例如：

```纯文本
print(eval_domain[100] ** 2)
half_domain_size = len(eval_domain) // 2
print(eval_domain[half_domain_size + 100] ** 2)
```

同样，第三层的定义域将是：

$$
w^4,(w\cdot h)^4,(w\cdot h^2)^4,...,(w\cdot h^{2047})^4
$$

以此类推，等等。

编写一个函数 `next_fri_domain` ，它将前一个定义域作为参数，并输出下一个。

```纯文本
def next_fri_domain(fri_domain):
    return [x ** 2 for x in fri_domain[:len(fri_domain) // 2]]
```

跑测试：

```纯文本
# Test against a precomputed hash.
from hashlib import sha256
next_domain = next_fri_domain(eval_domain)
assert '5446c90d6ed23ea961513d4ae38fc6585f6614a3d392cb087e837754bfd32797' == sha256(','.join([str(i) for i in next_domain]).encode()).hexdigest()
print('Success!')
```

### FRI 折叠操作

第一个FRI多项式只是组合多项式，即`cp`

每个后续 FRI 多项式由下式获得：

1.  获取随机域元素 $\beta$ (通过调用`Channel.receive_random_field_element`)
2.  将前一个多项式的奇系数乘以 $\beta$
3.  将系数的连续对（偶数-奇数）相加。

形式上，假设第 k 个多项式是次数小于 $m$ （对于某些 $m$ 是 2 的幂）:

$$
p_k(x):=\sum_{i=0}^{m-1}c_ix^i
$$

然后是第 (k+1) 个多项式，其次数将小于 $\frac m 2$ :

$$
p_{k+1}(x):=\sum_{i=0}^{m/2-1}(c_{2i}+\beta \cdot c_{2i+1})x^i
$$

编写一个函数 `next_fri_polynomial`，它将一个多项式和一个域元素（我们称为 $\beta$ ）作为参数，并返回“折叠”的下一个多项式。

注意：

1.  `Polynomial.poly `包含多项式系数的列表，自由项在前，最高次数在后，因此` p.poly[i] == u` 如果 $x^i$ 系数是 $u$ 。
2.  `Polynomial`的默认构造函数将系数列表作为参数。 因此，可以通过调用 `Polynomial(l)` 从系数列表`  l  `实例化多项式。

```纯文本
def next_fri_polynomial(poly,  beta):
    odd_coefficients = poly.poly[1::2]
    even_coefficients = poly.poly[::2]
    odd = beta * Polynomial(odd_coefficients)
    even = Polynomial(even_coefficients)
    return odd + even
```

跑测试：

```纯文本
next_p = next_fri_polynomial(cp, FieldElement(987654321))
assert '6bff4c35e1aa9693f9ceb1599b6a484d7636612be65990e726e52a32452c2154' == sha256(','.join([str(i) for i in next_p.poly]).encode()).hexdigest()
print('Success!')
```

### 将它们放在一起以获得下一个 FRI 层

编写一个函数 `next_fri_layer` ，将一个多项式，一个定义域，一个域元素 ( $\beta$ )作为参数，并返回下一个多项式，下一个定义域，和在下一个定义域上对下一个多项式的评估。

```纯文本
def next_fri_layer(poly, domain, beta):

  next_poly = next_fri_polynomial(poly, beta)

  next_domain = next_fri_domain(domain)

  next_layer = [next_poly(x) for x in next_domain]

  return next_poly, next_domain, next_layer
```

跑测试:

```纯文本
test_poly = Polynomial([FieldElement(2), FieldElement(3), FieldElement(0), FieldElement(1)])
test_domain = [FieldElement(3), FieldElement(5)]
beta = FieldElement(7)
next_p, next_d, next_l = next_fri_layer(test_poly, test_domain, beta)
assert next_p.poly == [FieldElement(23), FieldElement(7)]
assert next_d == [FieldElement(9)]
assert next_l == [FieldElement(86)]
print('Success!')
```

## 生成 FRI 承诺

我们现在已经开发了编写 `FriCommit` 方法的工具，其中包含主要的 FRI 承诺循环。

它需要以下 5 个参数：

1.  组合多项式，也就是第一个 FRI 多项式，即 - `cp`。
2.  8192 阶的陪集也是第一个 FRI 定义域，即 - `eval_domain`。
3.  前者对后者的评估，这也是第一个 FRI 层，即 - `cp_eval`。
4.  根据这些评估构建的第一棵 Merkle 树（我们将为每个 FRI 层创建一个 Merkle 树），即`cp_merkle`。
5.  一个信道对象，即`channel`。

该方法相应地返回 4 个列表：

1.  FRI多项式
2.  FRI 定义域
3.  FRI层
4.  FRI Merkle 树

该方法包含一个循环，在每次迭代中我们使用每个列表中的最后一个元素扩展这四个列表。 一旦最后一个 FRI 多项式的次数为 0，即 - 当最后一个 FRI 多项式只是一个常数时，迭代应该停止。 然后它应该通过信道发送这个常数（即 - 多项式的自由项）。 `Channel` 类仅支持发送字符串，因此请确保在发送之前将你希望通过信道发送的任何内容转换为字符串。

```纯文本
def FriCommit(cp, domain, cp_eval, cp_merkle, channel):    
    fri_polys = [cp]
    fri_domains = [domain]
    fri_layers = [cp_eval]
    fri_merkles = [cp_merkle]
    while fri_polys[-1].degree() > 0:
        beta = channel.receive_random_field_element()
        next_poly, next_domain, next_layer = next_fri_layer(fri_polys[-1], fri_domains[-1], beta)
        fri_polys.append(next_poly)
        fri_domains.append(next_domain)
        fri_layers.append(next_layer)
        fri_merkles.append(MerkleTree(next_layer))
        channel.send(fri_merkles[-1].root)   
    channel.send(str(fri_polys[-1].poly[0]))
    return fri_polys, fri_domains, fri_layers, fri_merkles
```

跑测试：

```纯文本
test_channel = Channel()
fri_polys, fri_domains, fri_layers, fri_merkles = FriCommit(cp, eval_domain, cp_eval, cp_merkle, test_channel)
assert len(fri_layers) == 11, f'Expected number of FRI layers is 11, whereas it is actually {len(fri_layers)}.'
assert len(fri_layers[-1]) == 8, f'Expected last layer to contain exactly 8 elements, it contains {len(fri_layers[-1])}.'
assert all([x == FieldElement(-1138734538) for x in fri_layers[-1]]), f'Expected last layer to be constant.'
assert fri_polys[-1].degree() == 0, 'Expacted last polynomial to be constant (degree 0).'
assert fri_merkles[-1].root == '1c033312a4df82248bda518b319479c22ea87bd6e15a150db400eeff653ee2ee', 'Last layer Merkle root is wrong.'
assert test_channel.state == '61452c72d8f4279b86fa49e9fb0fdef0246b396a4230a2bfb24e2d5d6bf79c2e', 'The channel state is not as expected.'
print('Success!')
```

运行以下代码块以使用你的信道对象执行该函数并打印到目前为止的证明：

```纯文本
fri_polys, fri_domains, fri_layers, fri_merkles = FriCommit(cp, eval_domain, cp_eval, cp_merkle, channel)
print(channel.proof) 
```
