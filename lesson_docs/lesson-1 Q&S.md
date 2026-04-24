
Q:npm install polkadot-api viem ethers @polkadot/util-crypto @polkadot/util typescript tsx解释这个在做什么
A:这条命令安装了 7 个 npm 包，每个的作用如下：

| 包名 | 作用 | 在作业中用于 |
|------|------|-------------|
| **polkadot-api** | Polkadot 官方新一代 JS API（简称 PAPI），用于通过 WebSocket 连接 Substrate 链，查询链上存储、发送交易 | Task 1 中通过 `api.query.System.Account` 查询 Substrate 原生余额 |
| **ethers** | 以太坊生态最流行的 JS 库，提供 Provider、Wallet、合约交互等 | Task 1 查询 EVM 余额，Task 2 调用预编译合约 |
| **viem** | 以太坊生态的另一个现代 JS 库（ethers 的替代品），更轻量、类型更安全 | 和 ethers 做同样的事，作业要求用两种库 |
| **@polkadot/util-crypto** | Polkadot 工具库，提供 SS58 地址编解码、哈希等密码学工具 | Task 1 中 `encodeAddress` / `decodeAddress` 做 SS58 ↔ 字节数组转换 |
| **@polkadot/util** | Polkadot 工具库，提供 `u8aToHex`、`hexToU8a` 等字节数组/hex 互转工具 | 配合 `@polkadot/util-crypto` 使用 |
| **typescript** | TypeScript 编译器 | 项目用 TypeScript 编写 |
| **tsx** | 直接运行 `.ts` 文件的工具（不用先编译成 JS） | `npm run task1` 实际执行的是 `tsx src/task1-address-conversion.ts` |

简单来说就是三组东西：
- **Substrate 侧**：`polkadot-api` + `@polkadot/util-crypto` + `@polkadot/util`
- **EVM 侧**：`ethers` + `viem`
- **开发工具**：`typescript` + `tsx`

Q:olkadot-api	Polkadot 官方新一代 JS API（简称 PAPI），用于通过 WebSocket 连接 Substrate 链，查询链上存储、发送交易这个是主要用来干什么的这个库
A:`polkadot-api` 是用来**直接跟 Substrate 链的原生层交互**的，可以理解为 **Polkadot 世界的 ethers.js**。

## 核心用途

它通过 **WebSocket** 连接到 Substrate 节点的原生 RPC（不是 EVM 的 JSON-RPC），能做以下事情：

### 1. 查询链上存储（最常用）
```typescript
// 查询某个账户的余额、nonce 等信息
const accountInfo = await api.query.System.Account.getValue("5Grw...");
console.log(accountInfo.data.free); // 原生代币余额
```

### 2. 发送交易（Extrinsic）
```typescript
// 转账
const tx = api.tx.Balances.transferKeepAlive({
  dest: "5Dest...",
  value: 1000000000000n,
});
await tx.signAndSubmit(signer);
```

### 3. 订阅链上事件
```typescript
// 实时监听新区块
client.finalizedBlock$.subscribe(block => {
  console.log("新区块:", block.number);
});
```

### 4. 调用 Runtime API
```typescript
// 调用链上的 runtime 方法
const metadata = await api.apis.Metadata.metadata();
```

## 为什么作业需要它？

作业要求验证 **同一个账户** 在 EVM 视角和 Substrate 视角下余额是否一致：

| 工具 | 连接方式 | 查的是什么 |
|------|---------|-----------|
| ethers / viem | HTTP → EVM JSON-RPC | EVM 层看到的余额 (`eth_getBalance`) |
| **polkadot-api** | **WebSocket → Substrate RPC** | **Substrate 原生层的余额** (`System.Account`) |

两者查的是同一条链（Westend Asset Hub）的同一个账户，只是从不同的"入口"去查，所以余额应该一致。

## 一句话总结

> **ethers/viem 走的是 EVM 的门，polkadot-api 走的是 Substrate 原生的门，两扇门进的是同一个房间。**

Q：@polkadot/util-crypto	Polkadot 工具库，提供 SS58 地址编解码、哈希等密码学工具这个是主要用来干什么的这个库
A:`@polkadot/util-crypto` 主要就是做 **Polkadot 地址格式转换**的工具箱。

## 在作业中只用了两个函数

```typescript
import { encodeAddress, decodeAddress } from "@polkadot/util-crypto";
```

### `encodeAddress` — 字节数组 → SS58 地址
```typescript
// 把 32 字节的 AccountId 编码成人类可读的 SS58 地址
encodeAddress(
  [0x70, 0x72, 0x05, ...],  // 32 字节
  42                         // SS58 前缀 (42 = 通用 Substrate)
)
// → "5Ec96GKnuFKGarKaQwf2tqoAceuaEnv5b4cH5We86z61jS8i"
```

### `decodeAddress` — SS58 地址 → 字节数组
```typescript
// 反过来，把 SS58 地址解码回 32 字节
decodeAddress("5Ec96GKnuFKGarKaQwf2tqoAceuaEnv5b4cH5We86z61jS8i")
// → Uint8Array(32) [0x70, 0x72, 0x05, ...]
```

## 类比理解

| 概念 | 以太坊 | Polkadot |
|------|--------|---------|
| 原始地址 | 20 字节 hex (`0x7072...`) | 32 字节 AccountId |
| 人类可读格式 | checksum hex (`0x7072...A00A`) | **SS58** (`5Ec96G...jS8i`) |
| 编码工具 | `ethers.getAddress()` | **`encodeAddress()`** |
| 解码工具 | `ethers.getBytes()` | **`decodeAddress()`** |

## SS58 是什么？

SS58 就是 Polkadot 的地址编码格式，类似 Bitcoin 的 Base58Check：

```
原始字节 → 加上网络前缀 → Base58 编码 → 加上校验码
```

不同网络有不同前缀，所以同一个公钥在不同链上显示不同地址：
- 前缀 `0` → Polkadot：`1xxx...`
- 前缀 `2` → Kusama：`Cxxx...`  
- 前缀 `42` → 通用/测试：`5xxx...`

## 一句话总结

> **这个库就是干 "SS58 地址 ↔ 原始字节" 互转的，作业里需要它是因为地址转换的核心步骤就是在字节层面操作，然后用它编解码成人类可读的 SS58 格式。**

Q：为什么要进行地址转换？？原始字节是怎么生成的，ss58地址SS58 就是 Polkadot 的地址编码格式，那为什么要和原始字节互相转换，地址的转换有哪些地址的互相转化
A：## 地址是怎么来的？

一切从**密钥对**开始：

```
私钥 (随机数)
  → 椭圆曲线运算 → 公钥 (原始字节)
                       → 编码 → 人类可读地址
```

| 链 | 公钥长度 | 编码方式 | 地址样子 |
|----|---------|---------|---------|
| Ethereum | 64字节 → keccak256 → 取后 **20字节** | Hex + Checksum | `0x7072...A00A` |
| Polkadot | **32字节** | SS58 编码 | `5Ec96G...jS8i` |

所以**原始字节就是公钥**（或公钥的哈希），地址只是它的"显示格式"。

## 为什么要转换？

因为 **Polkadot Asset Hub 上同时跑着两套系统**：

```
┌──────────────────────────────────────┐
│         Westend Asset Hub            │
│                                      │
│  ┌──────────────┐  ┌──────────────┐  │
│  │  Substrate层  │  │   EVM 层     │  │
│  │  (原生)      │  │ (pallet-     │  │
│  │              │  │  revive)     │  │
│  │  32字节地址   │  │  20字节地址   │  │
│  │  SS58 格式   │  │  0x hex 格式  │  │
│  └──────┬───────┘  └──────┬───────┘  │
│         │    同一个账户     │          │
│         └────────┬────────┘          │
│                  │                    │
│           需要地址映射！               │
└──────────────────────────────────────┘
```

**问题来了**：MetaMask 用的是 20 字节的 `0x` 地址，polkadot.js 用的是 32 字节的 SS58 地址。如果我在 MetaMask 里有余额，我怎么在 Substrate 层查到它？→ 需要**地址转换**。

## 作业中涉及的地址转换

一共有 **3 种地址格式**，两两之间需要转换：

```
  32字节 AccountId (原始字节)
       ↗                ↘
  SS58 地址              H160 地址
  (Substrate用)         (EVM用)
```

### 转换 1: SS58 ↔ 原始字节（纯编码转换）

这是最简单的，**信息不丢失**，就像 UTF-8 编码一样：

```typescript
// SS58 → 字节（解码）
decodeAddress("5Ec96G...") → [0x70, 0x72, ... 32字节]

// 字节 → SS58（编码）
encodeAddress([0x70, 0x72, ...], 42) → "5Ec96G..."
```

**为什么需要**：PAPI 查余额要传 SS58 字符串，但地址映射要在字节层面操作。

### 转换 2: H160 → 32字节 AccountId（EVM 地址映射到 Substrate）

20 字节不够 32 字节怎么办？**后面补 12 字节 `0xEE`**：

```
H160:          [0x70 0x72 ... 共20字节]
AccountId32:   [0x70 0x72 ... 共20字节] [0xEE 0xEE ... 共12字节]
```

```typescript
function h160ToAccountId32(h160: string) {
  const bytes = new Uint8Array(32);
  bytes.fill(0xEE);            // 先全填 0xEE
  bytes.set(addressBytes, 0);  // 前20字节放 H160
  return bytes;
}
```

**为什么需要**：你在 MetaMask 有个地址 `0x7072...`，想用 PAPI 查它的 Substrate 余额，就得先把它转成 SS58 → 需要先映射成 32 字节。

### 转换 3: 32字节 AccountId → H160（Substrate 地址映射到 EVM）

两种情况：

```typescript
function accountId32ToH160(accountId: Uint8Array) {
  if (后12字节全是0xEE) {
    // 情况A: 本来就是 EVM 地址 → 直接取前20字节
    return accountId.slice(0, 20);
  } else {
    // 情况B: 原生 Substrate 地址 → keccak256 哈希后取后20字节
    return keccak256(accountId).slice(12, 32);
  }
}
```

**为什么需要**：你有一个 Substrate 账户 `5Grw...`（Alice），想在 ethers 里查它的 EVM 余额，就得先转成 `0x` 地址。

## 完整流程图

```
MetaMask 钱包地址 0x7072...A00A (20字节 H160)
    │
    │  h160ToAccountId32()  补0xEE
    ▼
AccountId32 (32字节原始字节)
    │
    │  encodeAddress()  SS58编码
    ▼
SS58 地址 5Ec96G...jS8i
    │
    │  传给 PAPI
    ▼
api.query.System.Account.getValue("5Ec96G...")  → 余额
```

## 一句话总结

> **地址转换的本质就是：EVM 世界用 20 字节，Substrate 世界用 32 字节，两边要互通就必须有一套映射规则（补 0xEE / keccak256），而 SS58 只是 32 字节的"显示格式"。**

Q：papi是什么
A：## PAPI = Polkadot API

全称 **P**olkadot **API**，是 Polkadot 官方的**新一代 JavaScript/TypeScript SDK**，npm 包名是 `polkadot-api`。

### 它解决什么问题？

Polkadot 链不是以太坊，不能用 ethers/viem 去查原生数据。你需要一个专门的库来跟 Substrate 链"说话"：

```
以太坊生态:  ethers.js / viem    ← 通过 JSON-RPC 和链交互
Polkadot生态: polkadot-api (PAPI) ← 通过 WebSocket 和链交互
```

### 它能干什么？

| 功能 | 代码示例 |
|------|---------|
| 查余额 | `api.query.System.Account.getValue(address)` |
| 转账 | `api.tx.Balances.transferKeepAlive(...)` |
| 监听新区块 | `client.finalizedBlock$.subscribe(...)` |
| 查资产 | `api.query.Assets.Account.getValue(assetId, address)` |

### 为什么叫"新一代"？

之前 Polkadot 用的是 `@polkadot/api`（polkadot-js），但它有很多问题：包太大、类型不安全、API 笨重。PAPI 是它的替代品：

| | 旧版 `@polkadot/api` | 新版 `polkadot-api` (PAPI) |
|---|---|---|
| 包体积 | 很大 | 小很多 |
| 类型安全 | 弱 | **强类型**（从链元数据自动生成） |
| 维护状态 | 逐步废弃 | 官方推荐 |

### 作业中用了什么？

就一行核心代码：

```typescript
const api = client.getTypedApi(wah);  // wah = Westend Asset Hub

// 查某个 SS58 地址的余额
const info = await api.query.System.Account.getValue("5Ec96G...");
console.log(info.data.free);  // → 原生代币余额 (planck 单位)
```

### 为什么之前要跑 `npx papi add wah -n westend2_asset_hub`？

PAPI 的核心特性是**类型安全**。它需要先下载链的元数据，然后自动生成 TypeScript 类型文件，这样你写代码时就有完整的自动补全和类型检查：

```
npx papi add wah -n westend2_asset_hub
    │
    ├── 1. 连接到 Westend Asset Hub 节点
    ├── 2. 下载链的 metadata（都有哪些 pallet、存储、交易）
    ├── 3. 生成 TypeScript 类型到 .papi/descriptors/
    └── 4. 你就可以 import { wah } from "@polkadot-api/descriptors"
```

### 一句话总结

> **PAPI 就是 Polkadot 链的 ethers.js，专门用来跟 Substrate 原生层交互的 SDK。**

Q：那波卡的合约不是可以用solidity开发，又如何和papi结合
A：这是一个很好的问题！关键在于理解 **Polkadot 链的分层架构**。

## 一条链，两个入口

Westend Asset Hub 同时提供两套接口：

```
                    Westend Asset Hub
┌─────────────────────────────────────────────┐
│                                             │
│   ┌─────────────┐     ┌─────────────────┐  │
│   │  EVM 入口    │     │  Substrate 入口  │  │
│   │             │     │                 │  │
│   │  Solidity   │     │  Pallet 调用     │  │
│   │  合约交互    │     │  原生存储查询     │  │
│   │             │     │                 │  │
│   │  工具:       │     │  工具:           │  │
│   │  ethers     │     │  PAPI           │  │
│   │  viem       │     │                 │  │
│   │  Hardhat    │     │                 │  │
│   └──────┬──────┘     └────────┬────────┘  │
│          │                     │            │
│          ▼                     ▼            │
│   eth-rpc proxy          WebSocket RPC     │
│   (HTTP JSON-RPC)        (Substrate原生)    │
│          │                     │            │
│          └──────────┬──────────┘            │
│                     ▼                       │
│            pallet-revive (PolkaVM)          │
│            pallet-balances                  │
│            pallet-assets                    │
│            ... 其他 pallets ...              │
│                     ▼                       │
│              链的底层状态存储                  │
└─────────────────────────────────────────────┘
```

## 实际开发场景

### 场景 1：纯 EVM 操作 → 只用 ethers/viem

```typescript
// 写 Solidity 合约、部署、调用 → 和以太坊开发一模一样
const contract = new ethers.Contract(address, abi, signer);
await contract.transfer(to, amount);
```

### 场景 2：纯 Substrate 操作 → 只用 PAPI

```typescript
// 查原生资产、跨链转账 (XCM)、治理投票、Staking
await api.tx.Balances.transferKeepAlive({ dest, value });
```

### 场景 3：两者结合 → ethers + PAPI 一起用（作业就是这个）

这是最有趣的场景。举几个真实例子：

#### 例子 1：查 Solidity 合约用户的 Substrate 余额
```typescript
// 用户在 MetaMask 里操作你的 Solidity 合约
const evmAddress = "0x7072...";  // MetaMask 地址

// 你想在后端查这个用户的原生代币余额
// ethers 做不到 → 需要 PAPI
const ss58 = h160ToSs58(evmAddress);  // 地址转换
const balance = await api.query.System.Account.getValue(ss58);
```

#### 例子 2：给 Solidity 合约铸造原生资产

```typescript
// 用 PAPI 在 Substrate 层创建一个资产
await api.tx.Assets.create({ id: 100, admin: alice, min_balance: 1n });

// 这个资产自动映射到一个 EVM 预编译地址
// 用户就可以在 Solidity 合约里用 ERC-20 接口操作它
const erc20 = new ethers.Contract(assetAddress, erc20Abi, provider);
const balance = await erc20.balanceOf(userAddress);
```

#### 例子 3：Solidity 合约 + Substrate 治理

```typescript
// 1. 用 Solidity 写业务逻辑 (ethers 部署和交互)
// 2. 用 PAPI 发起链上提案来升级合约参数
await api.tx.Council.propose({ ... });
```

## 为什么需要结合？

| 能力 | ethers/viem | PAPI |
|------|:-----------:|:----:|
| 部署 Solidity 合约 | ✅ | ❌ |
| 调用合约方法 | ✅ | ❌ |
| 查 EVM 余额 | ✅ | ❌ |
| 查原生 Substrate 余额 | ❌ | ✅ |
| 创建原生资产 | ❌ | ✅ |
| 跨链转账 (XCM) | ❌ | ✅ |
| 治理 / Staking | ❌ | ✅ |
| 查链上存储 (任意 pallet) | ❌ | ✅ |

## 一句话总结

> **Solidity 合约用 ethers/viem 交互，但 Polkadot 链上有很多 EVM 碰不到的原生功能（资产管理、跨链、治理），这些就需要 PAPI。两者各管一半，结合使用才能发挥 Polkadot 的全部能力。**

Q：测试balance是否一致是什么意思
A：作业原文是：

> 编程实现地址的转换，**并测试 balance 是否一致**

意思就是：**同一个账户，从 EVM 入口查的余额和从 Substrate 入口查的余额，数字应该相同。**

## 具体做什么

```
同一个人的钱包 → 有两个"门"可以查余额

门 1 (EVM): ethers.provider.getBalance("0x7072...") → 余额 A
门 2 (Substrate): api.query.System.Account("5Ec96G...") → 余额 B

验证: A === B ?
```

这两个地址指的是**同一个账户**（通过地址转换关联起来的），所以查到的余额理论上应该一样。

## 为什么要验证这个？

为了证明你前面的**地址转换做对了**。

如果转换错了：
```
0x7072... → 错误的SS58地址 → 查到的是别人的余额 → 不一致 ❌
```

如果转换对了：
```
0x7072... → 正确的SS58地址 → 查到的是同一个账户 → 一致 ✅
```

## 作业代码里的验证逻辑

```typescript
// 1. 从 H160 转换得到 SS58
const ss58 = h160ToSs58("0x7072...");

// 2. 分别查余额
const evmBalance  = await ethersProvider.getBalance("0x7072...");  // EVM视角
const subBalance  = await papiApi.query.System.Account.getValue(ss58); // Substrate视角

// 3. 比较
evmBalance === subBalance.data.free  // → 应该相等
```

## 一句话总结

> **"测试 balance 是否一致" = 用地址转换证明 EVM 地址和 SS58 地址指向同一个账户，查余额来验证。余额一样说明转换正确，余额不一样说明转换有问题。**

Q：2. 选择一个precompile来调用这个是什么意思？
A：## Precompile 是什么？

Precompile（预编译合约）是**内置在链底层的特殊合约**，不是任何人部署的，而是链本身自带的。

你可以把它理解为**系统内置函数**：

```
普通合约:  你自己写 Solidity → 部署 → 有字节码 → 运行在虚拟机里
预编译合约: 链出生时就存在 → 没有字节码 → 直接在底层原生代码里执行 → 更快更省 gas
```

## 有哪些 Precompile？

EVM 标准定义了 9 个预编译合约，都放在固定地址：

| 地址 | 名称 | 干什么的 | 通俗理解 |
|------|------|---------|---------|
| `0x01` | ecRecover | 从签名恢复签名者地址 | "这个签名是谁签的？" |
| `0x02` | SHA-256 | 计算 SHA-256 哈希 | 算哈希 |
| `0x03` | RIPEMD-160 | 计算 RIPEMD-160 哈希 | 算哈希（另一种） |
| `0x04` | Identity | 原样返回输入 | 你给它啥，它返回啥 |
| `0x05` | ModExp | 大数模幂运算 | 密码学计算 |
| `0x06-0x08` | bn128 系列 | 椭圆曲线运算 | 零知识证明用 |
| `0x09` | Blake2 | 计算 Blake2 哈希 | 算哈希（又一种） |

## "调用"是什么意思？

就是像调用普通合约一样，向这些地址发送数据，得到结果：

```typescript
// 调用 Identity 预编译 (0x04)
// 给它 0xdeadbeef，它就返回 0xdeadbeef
const result = await provider.call({
  to: "0x0000000000000000000000000000000000000004",  // Identity 地址
  data: "0xdeadbeef",                                 // 输入
});
// result === "0xdeadbeef" ← 原样返回

// 调用 SHA-256 预编译 (0x02)
// 给它 "hello" 的 hex，它返回 SHA-256 哈希
const hash = await provider.call({
  to: "0x0000000000000000000000000000000000000002",  // SHA-256 地址
  data: "0x68656c6c6f",                               // "hello"
});
// hash === "0x2cf24dba..." ← SHA-256("hello") 的结果
```

## 作业要求

> 选择一个 precompile 来调用

就是说：**从上面那张表里挑一个（或几个），用 ethers/viem 写代码调用它，展示输入输出。**

作业代码里选了 3 个：

| 选择 | 为什么选 |
|------|---------|
| Identity (0x04) | 最简单，能直观展示"输入=输出" |
| SHA-256 (0x02) | 最实用，能用 Node.js 本地验证结果 |
| ecRecover (0x01) | 最核心，签名验证是区块链的基础 |

## 为什么 Precompile 存在？

因为有些操作如果用 Solidity 写，太慢太贵。比如 ecRecover 涉及椭圆曲线运算，用 Solidity 实现要几百万 gas，但用预编译只要 3000 gas。

```
Solidity 写 SHA-256:  ❌ 几十万 gas，还不一定对
预编译 SHA-256:       ✅ 60 gas，底层 C 代码执行，又快又准
```

## 一句话总结

> **Precompile 就是链自带的"系统工具函数"，住在固定地址，不需要部署。作业要求你选一个，用 ethers 或 viem 调它，证明你会用。**

Q：我可以理解为波卡的链上跑着两套系统，一套是原生的substrate，一套EVM的。原生的由于秘钥公私钥生成方式是SS58所以生成的是32为的地址，EVM是兼容solidity的H160的20位的密码，为了兼容Metamaster的20位地址和两套系统之间的账号通信问题，所以做了一个账号映射，于是就有原始地址字节码和substrate原生层的SS58格式的转换和EVM的H160的地址的转换。那么问题来了原始字节码的对应的账户是怎么生成的，波卡链生成的吗？EVM模式下可以支持solidity开发是因为EVM层用revive做了兼容吗？revive是怎么兼容的？substrate和EVM两种形式应该是波卡开发链的两套不同的接口系统，对接的是波卡的底层链是这样吗？MetaMask的地址到波卡上是要先转成H160的在转成原始字节码再转成SS58吗？为什么？映射到的同一个账户在哪里？哪里产生？是否有一个账户系统？还是说只是一个转换规则？
A：你的理解大方向是对的！我逐个回答你的问题。

## 1. 原始字节码账户是怎么生成的？

**不是链生成的，是用户本地生成的。**

```
用户本地:
  随机数(私钥) → 椭圆曲线算法 → 公钥 → 这就是"原始字节码"(AccountId)
```

两套体系用的**密码学算法不同**：

| | Substrate 原生 | EVM |
|---|---|---|
| 签名算法 | sr25519 / ed25519 | secp256k1 (和以太坊一样) |
| 公钥长度 | 32 字节 | 64 字节 → keccak256 → 取后 **20 字节** |
| 地址格式 | SS58 编码 (人类可读) | `0x` hex (人类可读) |

关键点：**链不用事先"创建账户"**，只要你有私钥，地址就存在了。第一次有人给你转钱时，链上才会出现你的记录。

## 2. pallet-revive 怎么兼容 Solidity？

```
传统以太坊:
  Solidity → 编译成 EVM 字节码 → 在 EVM 虚拟机里执行

Polkadot Asset Hub:
  Solidity → 编译成 YUL(中间语言) → revive 编译器再编译成 RISC-V 字节码 → 在 PolkaVM 里执行
```

**不是跑了一个真正的 EVM**，而是：

```
┌──────────────────────────────────────────┐
│  你写的 Solidity 合约                      │
│         ↓ solc 编译                       │
│  EVM 字节码 / YUL 中间表示                  │
│         ↓ revive 编译器 (核心!)            │
│  RISC-V 指令 (PolkaVM 能运行的格式)         │
│         ↓                                │
│  PolkaVM 执行 ← Substrate 原生虚拟机       │
└──────────────────────────────────────────┘
```

同时，外面套了一层 **eth-rpc proxy**，假装自己是以太坊节点：

```
MetaMask → eth_sendTransaction → eth-rpc proxy → 翻译成 Substrate 交易 → 链上执行
```

所以 MetaMask、Hardhat、Remix 都以为自己在跟以太坊交互，实际上底层在 Polkadot 上跑。

## 3. 两套接口 → 同一条链？

**是的，完全正确。**

```
        用户层
     ┌─────┴─────┐
  MetaMask    polkadot.js
  (EVM接口)    (Substrate接口)
     │             │
     ▼             ▼
  eth-rpc       WebSocket
  proxy         RPC
     │             │
     └──────┬──────┘
            ▼
     ┌─────────────┐
     │  同一条链     │
     │  同一个数据库  │ ← 就一份状态存储
     │  同一份余额   │
     └─────────────┘
```

两个入口最终读写的是**同一份链上状态**，所以余额才会一致。

## 4. MetaMask 地址的转换流程？

MetaMask 里的地址**已经是 H160 了**（20 字节），不需要"先转成 H160"：

```
MetaMask 地址: 0x7072...A00A  (这就是 H160，20 字节)
     │
     │  补 12 字节 0xEE (映射规则)
     ▼
AccountId32: [0x70,0x72,...,0xEE,0xEE,...] (32 字节)
     │
     │  SS58 编码 (只是换个显示格式)
     ▼
SS58 地址: 5Ec96G...jS8i
```

## 5. 核心问题：映射的账户在哪里？是规则还是系统？

**这是最关键的问题。答案是：纯数学转换规则，不是一个账户系统。**

```
链上存储的只有一个东西:
  AccountId32 (32字节) → { balance: 100, nonce: 5, ... }
```

链不关心你是从 MetaMask 来的还是从 polkadot.js 来的。它只认 **32 字节的 AccountId32**。

那这两套接口怎么做到"同一个账户"的？

```
EVM 接口收到: 0x7072... (20字节)
  → eth-rpc proxy 内部自动补 0xEE → 得到 32字节
  → 拿这 32 字节去查链上状态
  → 返回余额

Substrate 接口收到: 5Ec96G... (SS58)
  → 自动解码成 32 字节 (和上面完全一样的 32 字节!)
  → 拿这 32 字节去查链上状态
  → 返回余额

两次查的是同一个 32 字节 → 同一条记录 → 余额一致
```

**没有一张"映射表"存在链上**。`0xEE` 填充是一个**确定性数学规则**，任何人在任何地方算出来结果都一样。

### 图示

```
              链上存储 (唯一真相)
              ┌────────────────────────┐
              │ key: 32字节 AccountId   │
              │ val: {free: 100, ...}  │
              └───────────┬────────────┘
                          │
            ┌─────────────┼─────────────┐
            │             │             │
         H160+0xEE     SS58解码       直接32字节
            │             │             │
        MetaMask     polkadot.js     程序直接调用
```

## 总结

| 你的问题 | 回答 |
|---------|------|
| 原始字节码怎么生成？ | 用户本地从私钥推导，不需要链参与 |
| revive 怎么兼容？ | 把 Solidity 重新编译成 RISC-V 字节码，在 PolkaVM 里跑 |
| 两套接口对接同一条链？ | 是的，读写同一份状态 |
| MetaMask 地址转换流程？ | H160(已有) → 补0xEE → AccountId32 → SS58编码 |
| 映射账户在哪？ | **没有映射表**，是纯数学规则。链上只存 32 字节 AccountId，两个入口用不同方式转换到同一个 32 字节 |

Q：也就是说两套接口其实一套是波卡的合约api方式也就是substrate，一套是solidity的开发api形式表面是solidity其实写完后通过revive层编译成为了波卡的API调用的底层链还是波卡的。这样就兼容了以太坊的开发生态。最后metamask本身就是h160的20位的地址，需要转换成为32位的通过补0xEE就得到了原始字节码，然后在转换成SS58,用于波卡链上操作，这一步为啥要转换是因为在PAPI查余额的时候需要传的是SS58的字符串，而不是原始字节码是吗？那如果是走EVM查询余额需要转换成ss58吗？链上的查询地址是原始字节码吗？
A：你的理解完全正确！回答最后几个问题：

## 链上存的是什么？

**链上存的是 32 字节原始字节码（AccountId32）**，不是 SS58，也不是 H160。

```
链上数据库 (底层 key-value 存储):

key:   0x7072056494a815425895c743e50c37a1b232a00aEEEEEEEEEEEEEEEEEEEEEEEE  (32字节)
value: { free: 1000000, reserved: 0, nonce: 3 }
```

SS58 和 H160 都只是**传输/显示格式**，到了链内部统一转成 32 字节。

## 不同入口各需要什么格式？

```
查余额的三种方式:

┌─────────────────────────────────────────────────────────┐
│ 方式 1: ethers/viem (走 EVM 入口)                        │
│                                                         │
│   provider.getBalance("0x7072...A00A")                  │
│                        ↑ 传 H160 就行                    │
│                        │                                │
│   eth-rpc proxy 内部: H160 → 补0xEE → 32字节 → 查链上    │
│                       (你不用管，proxy 自动做)             │
├─────────────────────────────────────────────────────────┤
│ 方式 2: PAPI (走 Substrate 入口)                         │
│                                                         │
│   api.query.System.Account.getValue("5Ec96G...jS8i")    │
│                                      ↑ 传 SS58 字符串    │
│                                      │                  │
│   PAPI 内部: SS58 → 解码成 32字节 → 查链上               │
│              (你不用管，PAPI 自动做)                      │
├─────────────────────────────────────────────────────────┤
│ 方式 3: 如果直接操作底层 (极少用)                          │
│                                                         │
│   直接传 32 字节 Uint8Array                              │
└─────────────────────────────────────────────────────────┘
```

## 回答你的三个具体问题

**Q: 转 SS58 是因为 PAPI 要求传 SS58 字符串？**

✅ 对。PAPI 的接口约定接收 SS58 格式字符串，收到后内部自动解码成 32 字节去查链。

**Q: 走 EVM 查余额需要转 SS58 吗？**

❌ 不需要。EVM 入口直接传 H160（20 字节的 `0x` 地址），eth-rpc proxy 内部自动补 0xEE 转成 32 字节。

**Q: 链上查询地址是原始字节码吗？**

✅ 对。链底层只认 32 字节。SS58 和 H160 都是**各个入口的"方言"**，进了链内部统一翻译成 32 字节。

## 全景图

```
MetaMask                    polkadot.js
  │                              │
  │ 传 H160                      │ 传 SS58
  ▼                              ▼
eth-rpc proxy               Substrate RPC
  │                              │
  │ H160+0xEE→32字节              │ SS58 解码→32字节
  ▼                              ▼
  └──────────┬───────────────────┘
             ▼
      链底层: 用 32 字节查存储
      storage[32字节] → 余额数据
```

**每个入口负责把自己的"方言"翻译成 32 字节，所以作为开发者你只需要传该入口期望的格式就行。作业里做地址转换是为了证明你理解这个映射关系，并验证不同入口查到的是同一个账户。**