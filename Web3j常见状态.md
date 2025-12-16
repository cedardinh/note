# web3j 4.9.8 默认区块状态含义

在 web3j v4.9.8 中，`DefaultBlockParameterName` 枚举提供了几个字符串常量，用来在调用 `eth_getBalance`、`eth_call`、`eth_getLogs` 等 JSON‑RPC 方法时指定区块高度或区块状态。这些常量对应以太坊 JSON‑RPC 的“默认区块参数”。官网文档对各状态有如下解释：

## 主要区块状态

| 枚举值 | 关键字 | 官方解释 | 适用场景 |
| --- | --- | --- | --- |
| `EARLIEST` | `"earliest"` | web3.js 文档指出，`"earliest"` 代表创世区块（genesis block）[docs.web3js.org](https://docs.web3js.org/api/web3-core/class/Web3Config/#:~:text=,of%20work%20network%27s%20latest%20blocks)；Chainstack 文档同样强调这是链上可用的最早或创世区块[docs.chainstack.com](https://docs.chainstack.com/reference/ethereum-getblockbynumber#:~:text=,but%20have%20not%20yet%20been)。 | 当需要查询链上最早状态时使用，例如检查合约在最初状态下的余额。 |
| `LATEST` | `"latest"` | 官方文档将 `"latest"` 定义为“当前链上最新的区块”[docs.web3js.org](https://docs.web3js.org/api/web3-core/class/Web3Config/#:~:text=,of%20work%20network%27s%20latest%20blocks)。它对应 PoW 链上的 tip 或 PoS 链上的 head，可能会经历重组。 | 查询当前链的最新状态或执行一般查询时默认采用。 |
| `PENDING` | `"pending"` | `"pending"` 代表“正在构建的区块”或“待打包交易的区块”[docs.web3js.org](https://docs.web3js.org/api/web3-core/class/Web3Config/#:~:text=,of%20work%20network%27s%20latest%20blocks)。Chainstack 文档说明它表示当前内存池中的待处理交易形成的区块[docs.chainstack.com](https://docs.chainstack.com/reference/ethereum-getblockbynumber#:~:text=,been%20included%20in%20a%20block)。 | 用于模拟调用或查看未出块前内存池中的交易效果（例如 `eth_call`）。 |
| `FINALIZED` | `"finalized"` | 在 PoS 网络中，`"finalized"` 表示“被超过 2/3 验证者接受为正统的区块”[docs.web3js.org](https://docs.web3js.org/api/web3-core/class/Web3Config/#:~:text=,of%20work%20network%27s%20latest%20blocks)。以太坊基金会文章指出，创建冲突区块需要攻击者至少烧毁 1/3 的质押 ETH，因此被重组的可能性极低[blog.ethereum.org](https://blog.ethereum.org/2021/11/29/how-the-merge-impacts-app-layer#:~:text=Finalized%20Blocks%20%26%20Safe%20Head)。 | 查询极高安全性的区块状态，如跨链桥、清算操作等要求最终确定性时使用。 |
| `SAFE` | `"safe"` | `"safe"` 是 PoS 网络的新标签，指“已被信标链证明（justified）的安全头区块”[docs.web3js.org](https://docs.web3js.org/api/web3-core/class/Web3Config/#:~:text=,of%20work%20network%27s%20latest%20blocks)。它与最新头部接近，但在正常情况下会被包含进正统链中，因此比普通 `latest` 区块更不容易被重组[docs.web3js.org](https://docs.web3js.org/api/web3-core/class/Web3Config/#:~:text=%2A%20%60%22safe%22%60%20,of%20work%20network%27s%20latest%20blocks)。以太坊基金会的文章称这类区块已获得 >2/3 验证者的见证，被认为会最终被确认[blog.ethereum.org](https://blog.ethereum.org/2021/11/29/how-the-merge-impacts-app-layer#:~:text=Finalized%20Blocks%20%26%20Safe%20Head)。 | 当应用希望查询一个很可能不会被链重组撤销的最新状态时使用，比如用户界面显示数据，但又不要求最终终结性。 |

## `ACCEPTED` 状态

在 web3j 4.9.8 的 `DefaultBlockParameterName` 枚举中还有 `ACCEPTED`，但以太坊 JSON‑RPC 规范和 web3.js 等主流文档并未定义名为 `accepted` 的默认区块参数。经过查找，`accepted` 与 **Avalanche** 等基于 Snowman 共识的区块链相关：

- Avalanche 文档解释，虚拟机（如 TimeStampVM）生成的每个区块都会被 Snowman 共识引擎验证并最终“接受（accepted）”或“拒绝（rejected）”[build.avax.network](https://build.avax.network/docs/avalanche-l1s/timestamp-vm/blocks#:~:text=In%20particular%2C%20this%20relationship%20is,to%20have%20the%20following%20interface)。
- 只有 **被接受的区块** 才会应用到链的状态；官方说明区块链的状态是将创世区块到最后一个接受的区块顺序执行得到的[build.avax.network](https://build.avax.network/docs/avalanche-l1s/virtual-machines-index#:~:text=Each%20block%20in%20the%20blockchain,latest%20state%20of%20the%20blockchain)。当共识引擎调用 VM 的 `accept()` 方法时，它会将该区块的状态标记为已接受并将其设置为“最后接受的区块”[build.avax.network](https://build.avax.network/docs/avalanche-l1s/timestamp-vm/blocks#:~:text=accept)。

因此，`ACCEPTED` 标签可能用于特定实现（如 Avalanche C‑Chain 的某些接口）来表示“最后被共识层接受的区块”，即链上当前有效状态的最新区块。它并非以太坊规范的一部分，web3j 通过包含该枚举以支持这些链的特定查询。对于普通以太坊节点，这个标签不会起作用。

## 总结

| 枚举值 | 中文含义 | 来源 |
| --- | --- | --- |
| `EARLIEST` | 创世区块，即编号最小的区块[docs.web3js.org](https://docs.web3js.org/api/web3-core/class/Web3Config/#:~:text=,of%20work%20network%27s%20latest%20blocks) | web3.js API 文档、Chainstack 文档 |
| `LATEST` | 最新区块（链上当前头部）[docs.web3js.org](https://docs.web3js.org/api/web3-core/class/Web3Config/#:~:text=,of%20work%20network%27s%20latest%20blocks) | web3.js API 文档 |
| `PENDING` | 正在构建的区块，包括待打包交易[docs.web3js.org](https://docs.web3js.org/api/web3-core/class/Web3Config/#:~:text=,of%20work%20network%27s%20latest%20blocks)[docs.chainstack.com](https://docs.chainstack.com/reference/ethereum-getblockbynumber#:~:text=,been%20included%20in%20a%20block) | web3.js API 文档、Chainstack 文档 |
| `FINALIZED` | 经 PoS 共识超过 2/3 验证者确认的区块，几乎不会被重组[docs.web3js.org](https://docs.web3js.org/api/web3-core/class/Web3Config/#:~:text=,of%20work%20network%27s%20latest%20blocks)[blog.ethereum.org](https://blog.ethereum.org/2021/11/29/how-the-merge-impacts-app-layer#:~:text=Finalized%20Blocks%20%26%20Safe%20Head) | web3.js API 文档、以太坊基金会博客 |
| `SAFE` | 安全头区块，经信标链证明且几乎肯定会成为正统链的一部分[docs.web3js.org](https://docs.web3js.org/api/web3-core/class/Web3Config/#:~:text=%2A%20%60%22safe%22%60%20,of%20work%20network%27s%20latest%20blocks)[blog.ethereum.org](https://blog.ethereum.org/2021/11/29/how-the-merge-impacts-app-layer#:~:text=Finalized%20Blocks%20%26%20Safe%20Head) | web3.js API 文档、以太坊基金会博客 |
| `ACCEPTED` | （主要用于 Avalanche 等链）指被 Snowman 共识引擎接受的区块。链的状态由创世区块到最后接受的区块顺序执行得到[build.avax.network](https://build.avax.network/docs/avalanche-l1s/virtual-machines-index#:~:text=Each%20block%20in%20the%20blockchain,latest%20state%20of%20the%20blockchain)；共识引擎调用 `accept()` 时会标记区块为已接受并更新“最后接受的区块”[build.avax.network](https://build.avax.network/docs/avalanche-l1s/timestamp-vm/blocks#:~:text=accept)。 | Avalanche Builder Hub 文档 |

`ACCEPTED` 目前不是以太坊 JSON‑RPC 的标准区块参数，但在支持 Snowman 共识的链或特定节点实现中可能可用。其他五个标签——`earliest`、`latest`、`pending`、`finalized` 和 `safe`——是以太坊 JSON‑RPC 的通用默认参数，web3j 将其封装为枚举以便开发者使用。EOF

# Documentation

https://docs.web3js.org/api/web3-core/class/Web3Config/#:~:text=,of%20work%20network%27s%20latest%20blocks