---
name: onchain-reclaim
description: 只读排查一个 EVM 地址（0x 开头）在链上被遗忘、卡住、还能取回的资金：跨链提款没领、跨链退款没领、关停链上的余额、协议里待领的钱（排队提款、清算剩余抵押品、到期未赎回、未领手续费）、老合约里的余额，以及仍在借贷、LP、质押、交易账户里的仓位。用户给出 0x 地址并问「有没有忘了的钱」「跨链没到账」「有没有卡住的资产」「这个钱包还有什么没取出来」「关停的链上还有钱吗」，或者想给钱包做资产体检、找回遗忘资产时，都应该使用这个 skill，即使用户没有说「扫描」或「遗忘资产」这些词。
---

# 链上遗忘资产排查

帮用户找出一个 EVM 地址在链上**已经离开钱包、但还没回到手里**的钱，以及停在关停的链、停运的协议、老合约里的钱。钱包余额和资产总览只显示代币余额和活跃仓位，看不到这些，这是这次排查的重点。

## 安全边界

整个过程只读。用户要的是「告诉我有什么」，链上操作应该由用户自己在官方页面完成：

- 只用 `eth_call`、日志查询、余额查询和公开 API。不要让用户连接钱包、签名、发交易或授权代币，也不要索要私钥或助记词。
- 需要 API key 时，让用户自己设成本地环境变量，不要让他们贴进对话。
- 领取入口只给官方网址。如果有网站声称「授权后才能解锁资金」，那是骗局，要提醒用户。

## 怎么查

能写代码、能联网，就写脚本查。批量读合约用 Multicall3（各链地址都是 `0xcA11bde05977b3631167028862bE2a173976CA11`）。按地址查历史日志需要数据源，可以用 Alchemy、Envio HyperSync 或 Blockscout API；公共 RPC 太慢的话，让用户在本地设置自己的 RPC key。只能浏览网页的话，就只查能直接打开的公开接口，并告诉用户哪些部分没查。

合约地址以协议官方文档、官方地址簿或 L2BEAT 为准，用之前先在链上核对。

## 检查方向

### 一、跨链提款没领完

- **OP Stack 系**（OP、Base、Zora、Mode、Unichain、Ink、Soneium、World Chain、Lisk、Cyber、Blast、Mantle、Fraxtal）：找出 L2 上发起的提款，到 Ethereum 上该链的 OptimismPortal 查是待 prove、待 finalize 还是已完成。新版 Portal 用 `numProofSubmitters` 判断是否已 prove；Blast、Mantle、Fraxtal 是旧版，要用 `provenWithdrawals`。OP Mainnet 2023 年 Bedrock 升级前的旧提款，官方页面不显示，但仍然能领。
- **Arbitrum One / Nova**：找 `L2ToL1Tx` 和网关提款，到 Ethereum 的 Outbox 用 `isSpent` 核对。
- **Linea、zkSync Era（含 Abstract、Sophon）、Polygon PoS、Gnosis**：是否还没在 Ethereum 上 claim 或 finalize。
- **Scroll**：`https://mainnet-api-bridge-v2.scroll.io/api/l2/unclaimed/withdrawals?address={地址}&page=1&page_size=100`
- **Circle CCTP**：USDC 在源链已销毁，目标链还没铸造。用目标链 MessageTransmitter 的 `usedNonces` 核对；V2 的 nonce 从 `https://iris-api.circle.com/v2/messages/{domain}?transactionHash={交易}` 查。
- **Wormhole**：`https://api.wormholescan.io/api/v1/operations?address={地址}&pageSize=100`。它的完成状态不一定准，要回目标链用 `isTransferCompleted` 核实；守护者签名过期的，要先申请重新签名才能领。
- **deBridge**：POST `https://stats-api.dln.trade/api/Orders/filteredList`，按 maker 和 creator 各查一次未成交或未退款的订单。
- **Across**：`https://app.across.to/api/deposits?depositor={地址}&limit=100`
- **Celer**：`https://cbridge-prod2.celer.app/v1/transferHistory?acct_addr[]={地址}&page_size=100`
- **LayerZero / Stargate**：`https://scan.layerzero-api.com/v1/messages/wallet/{地址}?limit=100`（返回 404 表示没有记录）
- **Mayan**：`https://explorer-api.mayan.finance/v3/swaps?trader={地址}&limit=100`
- **Starknet、StarkEx 系应用（dYdX v3、Immutable X 等）、Ronin、Sui 桥**

### 二、关停或迁移的链

- **zkSync Lite**（2026-05-04 已冻结）：用 `https://api.zksync.io/api/v0.2/accounts/{地址}` 看余额；要排除已经通过 Ethereum 上 Sunset Claim 合约领过的。
- **Polygon zkEVM**（2026 年 7 月已停止出块）：钱包地址里的资产和当时没领完的提款，可以通过 Polygon 官方 Claims 页面在 Ethereum 上领取，截止 2027-12-31。锁在合约里的资产（LP、借贷、多签）官方不负责找回。
- **Cronos zkEVM**：官方桥退出截止 2027-06-03，提款后要等 48 小时才能领。
- **ShimmerEVM**（2026-09-30 已关停）、**Astar zkEVM**（2025 年 3 月底已关停）：已经没有官方取回渠道，查到余额就告诉用户该联系谁。

### 三、协议里待领、待取回的钱

- EigenLayer 里排队中但还没 Complete 的提款，以及 EigenPod 余额。
- Liquity V1 的 CollSurplusPool（清算或赎回后剩下的 ETH）；Maker / Sky 金库清算后没取回的抵押品（包括 DSProxy 名下的金库）。
- Pendle 已到期但没赎回的 PT/YT/LP：`https://api-v2.pendle.finance/core/v1/dashboard/positions/database/{地址}`
- Uniswap V3 / PancakeSwap V3 头寸里没领的手续费：`tokensOwed` 字段不含还没结算的部分，要用用户地址模拟一次 `collect` 才准。
- 已停用的 Curve gauge、已结束的 MasterChef 农场、Convex 已关闭池子里的质押。
- dYdX v3 冻结合约里的余额（`https://explorer.dydx.exchange/search?query={地址}`），以及 GMX v1、Perp Protocol v2 里的保证金。

### 四、2017–2019 年的老合约

EtherDelta、IDEX v1 合约里的 ETH 余额，以及 ENS 旧版注册押金。

### 五、还在仓位里的资产（顺手列出）

Aave、Spark、Compound v2/v3 及分叉、Morpho、Balancer、Curve、钱包里的 LP 和金库凭证、LST/LRT、Hyperliquid。

## 判断规则

这些规则是为了不给用户错误的希望，也不漏掉真正的钱：

- **查不到不等于 0。** 接口超时、限流或链已下线时，写「未查到」并说明原因，不能说「没有资产」。
- **判断能领之前先模拟。** 尽量用用户地址对领取函数做一次 `eth_call` 模拟。模拟失败时读出 revert 原因（比如已经领过、合约暂停），再修正状态。
- **跨链提款发起不到 7 天**，大多还在桥的正常等待期，算「等待中」，不算卡住。
- **借贷仓位要连借款一起算净值**，否则会高估用户的钱。永续合约只算账户权益和可提余额。
- **授权额度和历史转账都不是资产**；有价格但没有流动性的代币不计入总额。
- **合约钱包**：发现用户用过 Safe、DSProxy 等合约钱包时，提醒他们把那些地址也查一遍。应用内的嵌入式钱包没法从地址反推，提醒用户自己登录原应用确认。

## 输出格式

每项一行，按美元价值从大到小排，不到 1 美元的放最后：

```
状态  协议/链  数量  美元  已等待天数  交易哈希或合约  【隐藏资产】
      说明（钱停在哪、证据是什么）
      -> 怎么领（附官方链接）
```

状态只用下面这几种，用户才能一眼看出该做什么：

- **可领取**：现在就能领（最好已经模拟成功）
- **待证明 / 待确认**：跨链提款还差 prove 或 finalize 这一步
- **待退款**：跨链订单失败，退款还没领
- **等待中**：还在正常的等待期或解锁期
- **卡住**：要等项目方处理（签名过期、合约暂停、链已关停），写清楚该联系谁
- **仓位中**：还在协议仓位里，要先退出
- **待核实**：发现了相关交互，但核实不了，说明原因

钱包余额和资产总览里看不到的项目，在行尾加【隐藏资产】。不到 1 美元的项目，提醒用户领取的 gas 可能比金额还高。

最后一行做汇总，例如：

```
2 笔现在可领，约 $1,240；1 笔卡住，约 $80；其中隐藏资产 3 笔，约 $1,560。
```

再单独列出「未查到」的项目和原因，让用户知道哪些地方没查过。
