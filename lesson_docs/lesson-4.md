#  4 作业知识点总结



## 1.第四课在做什么

完成了一个典型的区块链开发最小闭环：

1. 编写一个 Solidity 智能合约 `SimpleStorage`
2. 使用 Hardhat 编译合约
3. 使用 `ethers.js v6` 连接本地 Hardhat 区块链节点
4. 查询区块和账户余额
5. 发起一笔原生 ETH 转账
6. 部署智能合约
7. 调用合约函数，读取和更新链上状态

覆盖的是“链下脚本如何与链上合约交互”的基础能力。

## 2. 项目结构与各文件作用

目录中的关键文件如下：

- `contracts/SimpleStorage.sol`
  作用：定义链上的智能合约逻辑。
- `scripts/main.ts`
  作用：用 TypeScript 编写链下交互脚本，负责连接节点、转账、部署合约、调用合约。
- `hardhat.config.ts`
  作用：Hardhat 配置文件，指定 Solidity 编译器版本。
- `package.json`
  作用：定义项目依赖与 Node.js 项目元信息。
- `tsconfig.json`
  作用：定义 TypeScript 编译行为和模块解析方式。
- `artifacts/contracts/SimpleStorage.sol/SimpleStorage.json`
  作用：Hardhat 编译后生成的合约构建产物，包含 ABI 和 bytecode。
- `readme.md`
  作用：说明运行步骤。

可以把整个项目理解为两部分：

- 链上部分：`SimpleStorage.sol`
- 链下部分：`main.ts`

学习区块链开发时，这种“链上 + 链下”双端协作是最核心的思维方式。

## 3. Solidity 合约知识点

文件：`contracts/SimpleStorage.sol`

### 3.1 SPDX License Identifier

```solidity
// SPDX-License-Identifier: MIT
```

这是 Solidity 文件常见的许可证声明。它告诉工具链和开发者该文件采用什么开源许可证。

### 3.2 pragma solidity

```solidity
pragma solidity ^0.8.24;
```

这表示该合约要求使用 `0.8.24` 及兼容版本的 Solidity 编译器。

知识点：

- `pragma` 用来声明编译版本要求
- `^0.8.24` 表示允许 `0.8.x` 中不低于 `0.8.24` 的版本
- 保持合约和 Hardhat 配置里的版本一致很重要

### 3.3 contract 关键字

```solidity
contract SimpleStorage {
```

`contract` 类似面向对象语言中的类，但它部署后运行在区块链上。

### 3.4 状态变量

```solidity
uint256 private value;
```

这里定义了一个状态变量 `value`。

知识点：

- `uint256` 是无符号 256 位整数，也是 Solidity 中最常用的整数类型
- `private` 表示该变量不能被其他合约直接访问
- 状态变量存储在链上，修改它通常需要发送交易并消耗 gas

### 3.5 event 事件

```solidity
event ValueChanged(uint256 newValue);
```

事件用于把合约执行过程中的重要信息写入交易日志。

作用：

- 前端或脚本可以监听事件
- 方便追踪链上状态变化
- 比在链上存更多数据更节省 gas

虽然本作业没有继续解析这个事件日志，但定义事件是非常标准的合约开发习惯。

### 3.6 写函数：修改状态

```solidity
function store(uint256 newValue) public {
    value = newValue;
    emit ValueChanged(newValue);
}
```

知识点：

- `public` 表示外部账户和其他合约都可以调用
- `store` 会修改状态变量 `value`
- 修改链上状态的函数不是 `view` 或 `pure`
- 调用这类函数时需要发送交易
- `emit` 用来触发事件

要点：

- 调用 `store()` 不只是“调用函数”，本质上是“发送一笔会改变链状态的交易”
- 这类调用需要 gas、需要签名、需要等待区块确认

### 3.7 读函数：读取状态

```solidity
function retrieve() public view returns (uint256) {
    return value;
}
```

知识点：

- `view` 表示函数只读，不修改状态
- `returns (uint256)` 表示返回一个 `uint256`
- 调用只读函数通常不需要发送交易，也不消耗链上 gas（本地 RPC 模拟执行）

这是区分两类合约调用的关键：

- 读操作：`call`
- 写操作：`transaction`

## 4. Hardhat 工具链知识点

文件：`hardhat.config.ts`

```ts
const config: HardhatUserConfig = {
  solidity: "0.8.24",
};
```

### 4.1 Hardhat 是什么

Hardhat 是以太坊开发框架，常用于：

- 编译 Solidity 合约
- 启动本地区块链开发节点
- 部署合约
- 调试交易与合约
- 生成构建产物

### 4.2 Solidity 编译版本配置

这里指定了：

- 合约编译器版本为 `0.8.24`

它需要与 `SimpleStorage.sol` 中的 `pragma solidity ^0.8.24;` 对应起来。

### 4.3 本地 Hardhat 节点

作业运行依赖：

```bash
npx hardhat node
```

这个命令会启动一个本地测试链，一般监听：

```text
http://localhost:8545
```

它会自动提供一组测试账户及私钥，方便开发和调试。

### 4.4 编译产物 artifacts

执行：

```bash
npx hardhat compile
```

后会生成 `artifacts` 目录，其中包含：

- ABI：合约接口描述，告诉链下程序有哪些函数、参数和返回值
- bytecode：部署到链上的字节码

本作业后续部署合约时，就是从 artifact 中读取 ABI 和 bytecode。

## 5. TypeScript 与 Node.js 配置知识点

文件：`package.json`、`tsconfig.json`

### 5.1 package.json 中的依赖

关键依赖如下：

- `hardhat`
  提供编译、节点、开发框架能力
- `@nomicfoundation/hardhat-toolbox`
  Hardhat 常用工具集合
- `ethers`
  用于链下与以太坊兼容链交互
- `typescript`
  TypeScript 编译支持
- `ts-node`
  允许直接运行 `.ts` 文件
- `@types/node`
  Node.js 类型定义

### 5.2 ESM 模块

`package.json` 中有：

```json
"type": "module"
```

这表示项目使用 ES Module 模式，而不是 CommonJS。

因此脚本运行时用的是：

```bash
npx ts-node --esm scripts/main.ts
```

### 5.3 tsconfig.json 的核心含义

```json
{
  "target": "ES2022",
  "module": "NodeNext",
  "moduleResolution": "NodeNext",
  "esModuleInterop": true,
  "strict": true
}
```

重点理解：

- `target: ES2022`
  输出面向较新的 JavaScript 运行时
- `module: NodeNext`
  使用 Node.js 新模块系统规则
- `moduleResolution: NodeNext`
  模块解析遵循 NodeNext 方式
- `esModuleInterop: true`
  便于兼容不同模块系统
- `strict: true`
  开启严格类型检查，是好的工程习惯

## 6. ethers.js v6 基础知识点

文件：`scripts/main.ts`

该脚本是本作业最重要的学习内容。

### 6.1 导入库

```ts
import { ethers } from "ethers";
import fs from "fs";
```

这里用到了两类能力：

- `ethers`：区块链交互
- `fs`：读取本地文件系统中的合约 artifact

### 6.2 异步主函数

```ts
async function main() {
```

区块链交互大量依赖 RPC 网络请求，因此几乎所有关键操作都是异步的，需要 `async/await`。

例如：

- 获取网络信息
- 获取区块高度
- 查询余额
- 发送交易
- 等待交易确认
- 部署合约

### 6.3 Provider：连接区块链节点

```ts
const provider = new ethers.JsonRpcProvider("http://localhost:8545");
```

`Provider` 是链下程序访问区块链的入口。

它的职责包括：

- 查询区块
- 查询交易
- 查询余额
- 查询 nonce
- 调用只读合约方法
- 广播已签名交易

这里使用的是 `JsonRpcProvider`，表示通过 JSON-RPC 协议访问节点。

### 6.4 获取网络信息

```ts
const network = await provider.getNetwork();
```

知识点：

- 区块链节点会暴露链 ID、网络名称等信息
- 在真实项目里，经常要确认当前连接的是本地链、测试网还是主网

### 6.5 Wallet：由私钥控制账户

```ts
const sender = new ethers.Wallet("...", provider);
const receiver = new ethers.Wallet("...", provider);
```

知识点：

- `Wallet` 代表一个可签名账户
- 传入私钥后，就能对交易进行签名
- 将 `provider` 绑定给 `Wallet` 后，该钱包既能签名，又能直接向链发送交易

需要重点理解：

- 地址只是账户标识
- 私钥才是控制权
- 知道地址不等于可以花费该账户资产

本作业直接写死私钥，只适合本地开发测试，绝不能用于生产环境。

### 6.6 查询区块高度

```ts
const blockNumber = await provider.getBlockNumber();
```

区块高度表示当前链已经增长到哪个区块。

这是最基础的链上公共数据查询之一。

### 6.7 查询余额

```ts
const senderBalance = await provider.getBalance(sender.address);
```

知识点：

- 余额通常以最小单位返回
- 以太坊原生资产的最小单位是 `wei`
- 1 ETH = `10^18 wei`

所以脚本中使用了：

```ts
ethers.formatEther(senderBalance)
```

作用是把 `wei` 转成更适合人阅读的 ETH 字符串。

### 6.8 单位转换

```ts
const transferAmount = ethers.parseEther("1.5");
```

`parseEther("1.5")` 表示把人类可读的 `1.5 ETH` 转换为链上实际使用的最小单位。

这是一个非常关键的习惯：

- 输入金额时常用 `parseEther`
- 展示金额时常用 `formatEther`

## 7. 发送交易知识点

### 7.1 发送原生 ETH 转账

```ts
const tx = await sender.sendTransaction({
  to: receiver.address,
  value: transferAmount,
});
```

这里发起的是一笔最基础的原生币转账交易。

交易对象中常见字段：

- `to`：接收方地址
- `value`：转账金额
- `nonce`：发送者交易序号
- `gasLimit`：最多允许消耗多少 gas
- `gasPrice` 或 EIP-1559 相关字段：手续费参数

本作业只显式指定了 `to` 和 `value`，其余参数由节点或库帮助处理。

### 7.2 等待交易确认

```ts
await tx.wait();
```

知识点：

- `sendTransaction()` 返回的是交易响应，不代表已经最终上链确认
- `wait()` 会等待交易被打包进区块
- 区块确认后，链上状态才真正发生变化

如果不等待确认，后面立即查询余额或合约状态，可能拿到旧数据。

### 7.3 交易与调用的区别

务必理解：

- 查询余额、读取合约 `view` 函数：通常是“调用”
- 转账、部署合约、修改状态变量：都是“交易”

区别体现在：

- 是否需要签名
- 是否消耗 gas
- 是否会写入区块
- 是否需要等待确认

## 8. 合约部署知识点

### 8.1 读取 artifact 文件

```ts
const artifactStr = fs.readFileSync("./artifacts/contracts/SimpleStorage.sol/SimpleStorage.json", "utf-8");
const artifact = JSON.parse(artifactStr);
```

这里读取了 Hardhat 编译后的合约信息。

重点理解 artifact 一般包含：

- `abi`
- `bytecode`
- 部分编译元数据

### 8.2 ContractFactory

```ts
const factory = new ethers.ContractFactory(artifact.abi, artifact.bytecode, receiver);
```

`ContractFactory` 是“合约工厂对象”，用于创建合约实例并发起部署交易。

它需要三个核心输入：

- ABI：告诉库合约接口长什么样
- bytecode：告诉链要部署什么代码
- signer：谁来签名并支付部署 gas

这里使用 `receiver` 作为部署者，因此部署交易由 `receiver` 账户发出。

### 8.3 部署合约

```ts
const simpleStorage = await factory.deploy();
await simpleStorage.waitForDeployment();
const contractAddress = await simpleStorage.getAddress();
```

这里包含几个关键动作：

1. `deploy()` 发起部署交易
2. `waitForDeployment()` 等待部署完成
3. `getAddress()` 获取部署后的合约地址

需要理解：

- 合约地址不是预先固定写死的
- 每次部署都会产生一个新的合约地址
- 合约部署本质上也是一笔交易

## 9. 合约交互知识点

### 9.1 创建合约实例

```ts
const contract = new ethers.Contract(contractAddress, artifact.abi, receiver);
```

`ethers.Contract` 的核心构成是：

- 合约地址
- ABI
- provider 或 signer

如果传入的是 `provider`：

- 适合只读调用

如果传入的是 `signer`（如 wallet）：

- 既能读，也能写

本作业传入 `receiver`，所以可以直接调用写函数 `store()`。

### 9.2 调用只读方法

```ts
const initialValue = await contract.retrieve();
```

因为 `retrieve()` 是 `view` 函数，所以这是一次只读调用，不需要发送交易。

### 9.3 调用写方法

```ts
const updateTx = await contract.store(42n, { nonce });
await updateTx.wait();
```

这里有三个重要知识点：

#### 9.3.1 写方法会生成交易

`store(42n)` 不是本地立即赋值，而是提交一笔交易到链上执行。

#### 9.3.2 BigInt 写法

```ts
42n
```

这是 JavaScript / TypeScript 的 `BigInt` 字面量写法。

在 ethers v6 中，很多与链上数值相关的参数都推荐直接使用 `bigint`。

原因：

- 链上整数通常可能超过 JavaScript `number` 的安全范围
- `bigint` 更适合表示大整数

#### 9.3.3 nonce 概念

```ts
const nonce = await provider.getTransactionCount(receiver.address);
```

`nonce` 是某个账户已发送交易的序号。

作用：

- 防止交易重放
- 确保同一账户的交易顺序
- 每发送一笔新交易，nonce 通常加 1

本作业在调用 `store()` 时手动传入 nonce，这说明作者已经接触到账户交易排序的概念。

虽然在很多简单场景里可以不手动指定 nonce，但理解它非常重要。

## 10. 这份作业背后的完整执行流程

从运行角度，可以把整个作业理解为下面这条链路：

1. 启动本地 Hardhat 节点
2. 节点生成测试账户和测试资金
3. 编译 `SimpleStorage.sol`
4. Hardhat 生成 ABI 和 bytecode
5. `main.ts` 通过 `JsonRpcProvider` 连接本地节点
6. 使用测试私钥构造 `Wallet`
7. 查询区块高度和账户余额
8. 发起一笔 ETH 转账
9. 从 artifact 中读取 ABI 和 bytecode
10. 使用 `ContractFactory` 部署合约
11. 创建 `Contract` 实例
12. 调用 `retrieve()` 读取状态
13. 调用 `store(42)` 更新状态
14. 再次调用 `retrieve()` 验证状态是否已变化

这是一个非常标准的 Web3 初学者实验流程。

## 11. 这份作业涉及的核心概念总表

可以把知识点按模块整理如下。

### 11.1 链上基础

- 智能合约
- 状态变量
- 函数可见性 `public` / `private`
- `view` 函数
- 事件 `event`
- 合约部署

### 11.2 区块链账户模型

- 地址
- 私钥
- 签名
- 余额
- nonce

### 11.3 交易模型

- 转账交易
- 合约部署交易
- 合约写操作交易
- 等待确认 `wait()`

### 11.4 链下交互

- JSON-RPC
- Provider
- Wallet / Signer
- ContractFactory
- Contract 实例

### 11.5 工具链

- Hardhat
- TypeScript
- ts-node
- Node.js 文件系统 `fs`
- artifacts / ABI / bytecode

## 12. 容易混淆或出错的地方

### 12.1 `view` 函数和写函数不要混淆

- `retrieve()` 是只读调用
- `store()` 是写交易

两者在执行方式、gas 消耗和是否需要签名上完全不同。

### 12.2 地址不等于钱包

很多初学者容易把“地址”和“钱包”混为一谈。

更准确地说：

- 地址是账户标识
- 私钥控制地址
- Wallet 是链下工具对象，用私钥来签名交易

### 12.3 余额单位不是直接 ETH

链上底层通常用最小单位处理金额，所以需要：

- `parseEther()` 把 ETH 文本转成底层单位
- `formatEther()` 把底层单位转回可读 ETH

### 12.4 合约调用依赖 ABI

链下脚本之所以能写：

```ts
contract.retrieve()
contract.store(42n)
```

本质上依赖的是 ABI 描述。如果 ABI 不匹配，就无法正确编码和解码调用数据。

### 12.5 部署合约必须有 bytecode

ABI 只描述接口，不能部署合约。部署还必须有 bytecode。

所以：

- 交互需要 ABI
- 部署需要 ABI + bytecode

### 12.6 私钥不能用于真实环境

本作业中的私钥来自本地 Hardhat 测试环境，适合学习，不适合真实网络。

生产环境中通常会用：

- 环境变量
- 助记词管理
- 硬件钱包
- 安全密钥托管服务

## 13. 从这份作业中应该掌握的能力

如果学完这份作业，你至少应该能独立解释下面这些问题：

1. `Provider` 和 `Wallet` 分别负责什么？
2. 为什么转账和调用 `store()` 都需要等待交易确认？
3. 为什么读取合约需要 ABI？
4. 为什么部署合约需要 bytecode？
5. `retrieve()` 和 `store()` 的执行方式有什么差别？
6. `parseEther()` 与 `formatEther()` 分别解决什么问题？
7. 为什么链上整数经常使用 `bigint`？
8. nonce 为什么重要？

如果这些问题都能说清楚，说明你已经真正理解了这份作业，而不是只会照着命令运行。

### 13.1 问题答案

#### 13.1.1 Provider 和 Wallet 分别负责什么？

`Provider` 负责连接区块链节点，是读取链上数据和向节点发送请求的入口。比如查询区块高度、查询账户余额、获取 nonce、读取合约状态，都是通过 `Provider` 完成的。

`Wallet` 代表一个由私钥控制的账户，核心职责是签名交易。只有掌握私钥的钱包才能发起转账、部署合约、调用会修改状态的合约函数。把 `Wallet` 和 `Provider` 绑定后，这个钱包既能签名，也能把签名后的交易发送到链上。

简单理解：

- `Provider` 负责“连链、读链、发请求”
- `Wallet` 负责“控制账户、签名交易”

#### 13.1.2 为什么转账和调用 store() 都需要等待交易确认？

因为转账和 `store()` 都会改变链上状态。

转账会改变账户余额，`store()` 会改变合约中的状态变量 `value`。这类操作不是本地立即生效，而是先生成一笔交易，发送到节点，再等待矿工或验证者把交易打包进区块。

`sendTransaction()` 或 `contract.store()` 返回交易对象时，只表示交易已经被提交，不表示交易已经上链成功。只有执行 `await tx.wait()` 后，才能确认交易已经被打包，链上状态已经更新。

如果不等待确认就立刻查询余额或读取 `retrieve()`，可能读到旧状态。

#### 13.1.3 为什么读取合约需要 ABI？

链下程序不知道 Solidity 函数的名字、参数类型和返回值格式。ABI 就是合约对外暴露的接口说明。

读取合约时，`ethers.js` 需要根据 ABI 做两件事：

1. 把 `retrieve()` 这样的函数调用编码成 EVM 能理解的调用数据
2. 把链上返回的二进制数据解码成 JavaScript / TypeScript 能使用的结果

所以 ABI 的作用不是存储合约代码，而是告诉链下程序“这个合约能怎么调用，以及返回值该怎么解释”。

#### 13.1.4 为什么部署合约需要 bytecode？

部署合约时，链上需要真正执行和保存的是合约编译后的 EVM 字节码，也就是 `bytecode`。

Solidity 源码不能直接部署到链上，必须先经过编译，生成 EVM 可以执行的 bytecode。部署交易的核心内容就是把这段 bytecode 发送到链上，节点执行部署后，才会生成一个新的合约地址。

ABI 只描述接口，不能代表合约逻辑；bytecode 才是实际部署到链上的程序。

#### 13.1.5 retrieve() 和 store() 的执行方式有什么差别？

`retrieve()` 是 `view` 函数，只读取状态，不修改链上数据。调用它通常是一次本地 RPC 模拟执行，不需要签名，不会产生交易，也不会消耗链上 gas。

`store()` 会修改状态变量 `value`，所以必须发送交易。它需要账户签名，需要支付 gas，需要等待交易被打包确认。执行成功后，新的 `value` 才真正写入链上。

对比来看：

- `retrieve()`：读操作，调用 call，不改状态
- `store()`：写操作，发送 transaction，会改状态

#### 13.1.6 parseEther() 与 formatEther() 分别解决什么问题？

链上金额底层使用最小单位 `wei`，而人通常习惯用 `ETH`。

`parseEther()` 用来把人类可读的 ETH 数量转换成链上使用的 wei。例如：

```ts
ethers.parseEther("1.5")
```

表示把 `1.5 ETH` 转成 `1500000000000000000 wei`。

`formatEther()` 用来把链上返回的 wei 转成人类可读的 ETH 字符串。例如查询余额时，节点返回的是 wei，用 `formatEther()` 后才方便展示成 ETH。

简单理解：

- `parseEther()`：输入金额时用，ETH 转 wei
- `formatEther()`：展示金额时用，wei 转 ETH

#### 13.1.7 为什么链上整数经常使用 bigint？

区块链中的整数通常非常大，例如 `uint256` 最大可以表示 256 位无符号整数，远远超过 JavaScript `number` 的安全整数范围。

JavaScript 的 `number` 只能安全表示到 `Number.MAX_SAFE_INTEGER`，超过后可能出现精度丢失。金额、余额、gas、nonce、合约中的 `uint256` 都不能接受精度错误。

因此 ethers v6 经常使用 `bigint` 表示链上整数。`bigint` 可以安全表示任意大小的整数，更适合处理链上数值。

本作业中的：

```ts
contract.store(42n, { nonce })
```

这里的 `42n` 就是 JavaScript 的 `bigint` 写法。

#### 13.1.8 nonce 为什么重要？

nonce 是某个账户已经发送交易的序号。每个账户的交易 nonce 都按顺序递增。

nonce 主要有三个作用：

1. 防止同一笔交易被重复执行
2. 确定同一个账户发出的多笔交易的执行顺序
3. 让节点知道一笔交易是否是下一笔有效交易

如果 nonce 太小，交易可能会被认为已经用过；如果 nonce 太大，交易可能会一直等待前面的 nonce 交易先被处理。

所以在连续发送多笔交易、替换交易、或者手动控制交易顺序时，nonce 非常重要。本作业中通过：

```ts
const nonce = await provider.getTransactionCount(receiver.address);
```

获取 `receiver` 当前应该使用的下一笔交易序号，然后传给 `store()` 调用。

## 14. 建议的学习顺序

建议按下面顺序复习：

1. 先看 `SimpleStorage.sol`
   目标：理解状态变量、读写函数、事件
2. 再看 `scripts/main.ts` 前半部分
   目标：理解 provider、wallet、余额查询、转账
3. 再看 `main.ts` 后半部分
   目标：理解 artifact、ABI、bytecode、部署、合约调用
4. 最后看 `hardhat.config.ts` 和 `readme.md`
   目标：理解工具链是如何把整个流程串起来的

## 15. 后续可以继续扩展的练习

为了把这份作业真正学扎实，建议继续做下面这些练习：

1. 给 `SimpleStorage` 增加一个构造函数，在部署时传入初始值
2. 给合约增加 `owner` 变量，并限制只有 owner 才能调用 `store`
3. 在脚本中解析 `ValueChanged` 事件日志
4. 把私钥改成从 `.env` 文件读取
5. 为合约编写单元测试，而不是只写交互脚本
6. 尝试部署多个合约实例，观察地址变化
7. 故意发送多笔交易，观察 nonce 的变化

## 16. 一句话总结

这份作业的本质，是学习如何使用 `ethers.js` 让一个 TypeScript 脚本真正连接区块链、控制账户、发送交易、部署合约，并完成一次完整的链上状态读写闭环。
