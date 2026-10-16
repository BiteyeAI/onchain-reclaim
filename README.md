# onchain-reclaim

找回你在链上忘掉的钱。

给 AI Agent 用的 Skill。你发一个钱包地址，它会帮你查出这个地址里**已经离开钱包、却还没回到手里**的钱，还有留在已关停的链、停运的协议、老合约里的资产。

这些钱有个共同点：钱包余额和一般的资产总览里都看不到。

## 能找到什么

**跨链提款没领完**
跨链提款发起了，但最后一步 prove、finalize 或 claim 没做，钱就一直停在桥里。覆盖：
- 官方桥：OP Stack 系（OP、Base、Zora、Mode、Unichain、Ink、Soneium、World Chain、Lisk、Cyber、Blast、Mantle、Fraxtal）、Arbitrum、Linea、zkSync Era、Polygon PoS、Gnosis、Scroll
- 跨链协议：Circle CCTP、Wormhole、deBridge、Across、Celer、LayerZero / Stargate、Mayan
- OP Mainnet 2023 年升级前的旧提款，官方页面已经不显示了，但还能领

**关停或迁移的链**
zkSync Lite、Polygon zkEVM、Cronos zkEVM 上的余额，以及各自的领取方式和截止时间。

**协议里待领的钱**
- EigenLayer 排队中、还没完成的提款
- Liquity、Maker 清算后剩下的抵押品
- Pendle 到期后没赎回的仓位
- Uniswap V3 没领的手续费
- 已停用的农场和 gauge 里的质押

**2017–2019 年的老合约**
EtherDelta、IDEX v1 里的 ETH，ENS 旧版注册押金。

**还在仓位里的资产**
借贷、LP、质押、交易账户，也会顺带列出来，借贷会扣掉借款按净值算。

## 安装

需要一个能写代码、能联网的 AI Agent，比如 Claude Code。

**Claude Code**

```bash
git clone https://github.com/BiteyeAI/onchain-reclaim ~/.claude/skills/onchain-reclaim
```

装好后重启 Claude Code 即可。

**其他 Agent**

把 [SKILL.md](SKILL.md) 放进你的 Agent 的 skills 目录；不支持 skills 的话，也可以把全文复制到对话里。

## 使用

直接跟 Agent 说，比如：

> 帮我查一下 0x… 这个地址有没有忘了领的钱

> 我之前从 Base 跨到主网，好像一直没到账，帮我看看

> 这个钱包还有什么钱没取出来？

不用说 "skill" 这个词，Agent 会自己判断要不要用。

## 输出示例

```
可领取  Arbitrum One 官方桥  0.85 ETH  $2,075  412天  0x3f2a…  【隐藏资产】
        提款已过挑战期，但还没在 Ethereum 上执行 Outbox 领取
        -> 到 bridge.arbitrum.io 的交易记录里点 Claim
待确认  Base 官方桥  1,200 USDC  $1,200  95天  0x9c1e…  【隐藏资产】
        已 prove，还差 finalize
        -> 到 superbridge.app/base 的提款记录里点 Finalize
仓位中  Aave v3 · Ethereum  存款 2.1 ETH / 借款 1,500 USDC  $3,620
        -> 先还清借款，再到 Aave 取回存款

1 笔现在可领，约 $2,075；0 笔卡住，约 $0；其中隐藏资产 2 笔，约 $3,275。
```

> 以上为格式示例，数据是虚构的。

## 安全说明

- **全程只读。** 只读取链上数据和公开接口，不需要连接钱包，不签名，不发交易。
- **不要把私钥、助记词交给任何人**，包括 AI Agent。
- **领取一定要去官方页面。** 任何让你「先授权才能解锁资金」的网站都是骗局。
- **API key 只放本地。** Agent 可能会建议你设置一个 RPC 或数据服务的 key 来加快查询。请在自己电脑上设成环境变量，不要贴进对话。

## 局限

- 只支持 EVM 地址（0x 开头）。从 EVM 地址跨到 Solana 等链的转账可以查到，但不能直接输入 Solana 地址来查。
- Starknet、StarkEx 系应用、Ronin、Sui 桥的检查比较复杂，Agent 不一定能完整覆盖，结果里会标「未查到」。
- 查询结果依赖公开接口和 RPC。接口超时或限流时会标「未查到」，不代表没有资产。
- 美元价值是估算的，领取前请以官方页面为准。

## 免责声明

本项目只提供信息查询，不构成投资建议。链上操作有风险，请自行核实后再操作。
