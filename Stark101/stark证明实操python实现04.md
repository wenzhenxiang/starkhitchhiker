# stark证明实操python实现04

# 第四部分: 查询阶段

-   视频教程[ (youtube)](https://www.youtube.com/watch?v=CxP28qM4tAc " (youtube)")
-   幻灯片[(PDF)](https://starkware.co/wp-content/uploads/2021/12/STARK101-Part4.pdf "(PDF)")

### 加载之前代码

运行下面代码块以加载我们将在这部分中使用的变量。由于它会重复前面部分中所做的所有操作 - 运行需要一段时间。

```纯文本
from channel import Channel
from tutorial_sessions import part1, part3 

_, _, _, _, _, _, _, f_eval, f_merkle, _ = part1()
fri_polys, fri_domains, fri_layers, fri_merkles, _ = part3()

print('Success!')
```

## 查询上解承诺（decommit）

我们在这部分的目标是生成验证前三个部分的承诺所需的所有信息。 这部分我们写了两个函数：

1.  `decommit_on_fri_layers` - 以特定索引采样时，通过信道发送数据显示每个 FRI 层与其他层一致。
2.  `decommit_on_query` - 发送轨迹上解承诺所需的数据，然后调用 `decommit_on_fri_layers`。

### 在 FRI 层上解承诺

实施 `decommit_on_fri_layers` 函数。 该函数获取索引和信道，并通过信道发送相关数据以验证 FRI 层的正确性。 更具体地说，它迭代 `fri_layers` 和 `fri_merkles` 并且在每次迭代中它发送以下数据（按规定的顺序）：

1.  给定索引处的 FRI 层元素（使用 `fri_layers`）。
2.  它的身份验证路径（使用来自 `fri_merkles` 的相应 Merkle 树）。
3.  元素的 FRI 兄弟（即如果元素是 $cp_i(x)$ ，那么它的兄弟是 $cp_i(-x)$ ，  $cp_i$ 是当前层的多项式，并且 $x$ 是来自当前图层域的元素）。
4.  兄弟的身份验证路径（使用相同的 merkle 树）。

要获取元素的验证路径，请使用 `MerkleTree` 类的 `get_authentication_path() `，每次使用相应的索引。 请注意，元素的兄弟的索引等于 ( $idx + \frac k 2$ ) 模式， 其中 $k$ 是相关 FRI 层的长度。

请注意，我们**不**发送最后一层元素的身份验证路径。 在最后一层，所有元素都是相等的，无论查询如何，因为它们是常数多项式的评估。

（请记住在通过信道发送之前将非字符串变量转换为字符串。）

```纯文本
def decommit_on_fri_layers(idx, channel):
    for layer, merkle in zip(fri_layers[:-1], fri_merkles[:-1]):
        length = len(layer)
        idx = idx % length
        sib_idx = (idx + length // 2) % length        
        channel.send(str(layer[idx]))
        channel.send(str(merkle.get_authentication_path(idx)))
        channel.send(str(layer[sib_idx]))
        channel.send(str(merkle.get_authentication_path(sib_idx)))       
    channel.send(str(fri_layers[-1][0]))
```

跑测试：

```纯文本
# Test against a precomputed hash.
test_channel = Channel()
for query in [7527, 8168, 1190, 2668, 1262, 1889, 3828, 5798, 396, 2518]:
    decommit_on_fri_layers(query, test_channel)
assert test_channel.state == 'ad4fe9aaee0fbbad0130ae0fda896393b879c5078bf57d6c705ec41ce240861b', 'State of channel is wrong.'
print('Success!')
```

### 在轨迹多项式上解承诺

为了证明我们解承诺的 FRI 层确实是从组合多项式的评估中生成的，我们还必须发送：

1.  值 $f(x)$ 及其身份验证路径。
2.  值 $f(gx)$ 及其身份验证路径。
3.  值 $f(g^2x)$ 及其身份验证路径。

验证者知道组合多项式的随机系数，可以计算其在 $x$ 处的评估，并将其与第一个 FRI 层发送的第一个元素进行比较。

因此，函数 `decommit_on_query` 应该通过信道发送上述（1、2 和 3），然后调用 `decommit_on_fri_layers`。

重要的是，尽管 $x,gx,g^2x$ 是连续的元素（对群大小为 $|G|$ 取模) 在轨迹中，这些点中的 `f_eval` 的评估实际上相隔 8 个元素。 这样做的原因是我们在第一部分中将轨迹“放大”到其大小的 8 倍，以获得 Reed Solomon 码字。

提示：`f_eval`是组合多项式的评估，`f_merkle`是对应的Merkle树。

```纯文本
def decommit_on_query(idx, channel): 
    assert idx + 16 < len(f_eval), f'query index: {idx} is out of range. Length of layer: {len(f_eval)}.'
    channel.send(str(f_eval[idx])) # f(x).
    channel.send(str(f_merkle.get_authentication_path(idx))) # auth path for f(x).
    channel.send(str(f_eval[idx + 8])) # f(gx).
    channel.send(str(f_merkle.get_authentication_path(idx + 8))) # auth path for f(gx).
    channel.send(str(f_eval[idx + 16])) # f(g^2x).
    channel.send(str(f_merkle.get_authentication_path(idx + 16))) # auth path for f(g^2x).
    decommit_on_fri_layers(idx, channel)   
```

### 一组查询的解承诺

为了完成证明，证明者从信道中获取一组随机查询，即 0 到 8191 之间的索引，并在每个查询上解承诺。

使用你刚刚实现的函数 `decommit_on_query()` 和 `Channel.receive_random_int` 生成 3 个随机查询并在每个查询上解承诺。

```纯文本
def decommit_fri(channel):
    for query in range(3):
        # Get a random index from the verifier and send the corresponding decommitment.
        decommit_on_query(channel.receive_random_int(0, 8191-16), channel)

```

证明时间！

运行以下将它们联系在一起的代码块，运行所有以前的代码，以及你在这部分中编写的函数，并打印证明。

```纯文本
import time
from tutorial_sessions import part1, part3 

start = time.time()
start_all = start
print("Generating the trace...")
_, _, _, _, _, _, _, f_eval, f_merkle, _ = part1()
print(f'{time.time() - start}s')
start = time.time()
print("Generating the composition polynomial and the FRI layers...")
fri_polys, fri_domains, fri_layers, fri_merkles, channel = part3()
print(f'{time.time() - start}s')
start = time.time()
print("Generating queries and decommitments...")
decommit_fri(channel)
print(f'{time.time() - start}s')
start = time.time()
print(channel.proof)
print(f'Overall time: {time.time() - start_all}s')
print(f'Uncompressed proof length in characters: {len(str(channel.proof))}')
```
