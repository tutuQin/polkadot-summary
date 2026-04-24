
# Lesson 3 知识点总结

本文档用于总结 `course/lesson-3` 目录中涉及的主要知识点。该目录包含多个示例工程，覆盖 TypeScript 链下交互、Hardhat 合约开发、Hardhat Polkadot/PVM 开发，以及 Foundry 工具链。

## 1. 目录整体结构

`course/lesson-3` 主要包含四个部分：

- `local-dev`
  使用 TypeScript 和 `ethers.js` 连接本地 RPC，查询余额并发送转账。
- `hardhat-example`
  标准 Hardhat 示例工程，包含 Solidity 合约、测试、部署模块和 Polkadot Hub Testnet 配置。
- `hardhat-pvm-example`
  使用 `@parity/hardhat-polkadot` 的 Polkadot PVM 示例工程，合约是一个基于 OpenZeppelin 的 ERC20 Token。
- `my-foundry-project`
  Foundry 示例工程，包含 Solidity 合约、测试和部署脚本。

这节课的核心目标是理解：如何使用不同工具链完成智能合约的编写、测试、部署、验证和链下交互。

## 2. local-dev：TypeScript 与 ethers.js 链下交互

相关文件：

- `local-dev/src/index.ts`
- `local-dev/package.json`
- `local-dev/tsconfig.json`
- `local-dev/readme.md`

### 2.1 JsonRpcProvider

```ts
const provider = new JsonRpcProvider(url);
```

`JsonRpcProvider` 用来连接区块链 RPC 节点。脚本中默认连接：

```ts
const url = "http://localhost:8545";
```

这表示程序会通过本地 RPC 与链交互。

Provider 的主要职责包括：

- 查询账户余额
- 查询区块信息
- 查询交易信息
- 获取链上状态
- 广播签名后的交易

### 2.2 Wallet

```ts
const wallet = new ethers.Wallet(privateKey, provider);
```

`Wallet` 表示一个由私钥控制的账户。它的核心职责是签名交易。

把 `Wallet` 和 `Provider` 绑定后，这个钱包既可以签名，也可以把签名后的交易发送到链上。

需要注意：示例中的私钥只适合本地开发或测试环境，真实项目不能把私钥硬编码在代码中。

### 2.3 查询余额

```ts
const balance = await provider.getBalance(wallet.address);
console.log(ethers.formatEther(balance));
```

链上余额底层单位通常是 `wei`，不是 ETH。`formatEther()` 用来把 wei 转换成人类可读的 ETH 字符串。

### 2.4 发送转账交易

```ts
const tx = await wallet.sendTransaction({
    to: toAddress,
    value: ethers.parseEther("0.001"),
});
```

这里发起一笔原生币转账交易。

关键字段：

- `to`：接收方地址
- `value`：转账金额

`parseEther("0.001")` 会把 `0.001 ETH` 转换成链上使用的 wei。

### 2.5 等待交易确认

```ts
const receipt = await tx.wait();
```

`sendTransaction()` 返回交易响应，不代表交易已经上链。`tx.wait()` 会等待交易被打包进区块，并返回交易回执。

如果不等待确认，后续查询可能读到旧状态。

## 3. 本地 Polkadot Revive 开发环境

相关文件：

- `local-dev/readme.md`

### 3.1 revive-dev-node

`revive-dev-node` 是用于本地开发的 Polkadot Revive 节点。

示例编译命令：

```bash
cargo build -p revive-dev-node --bin revive-dev-node --release
```

### 3.2 eth-rpc

`eth-rpc` 提供以太坊兼容的 RPC 接口，使 `ethers.js`、Hardhat 等 EVM 工具可以与 Polkadot Revive 环境交互。

示例编译命令：

```bash
cargo build -p pallet-revive-eth-rpc --bin eth-rpc --release
```

### 3.3 启动本地节点和 RPC

```bash
./target/release/revive-dev-node --dev --tmp
./target/release/eth-rpc --dev
```

常见调试方式：

```bash
RUST_LOG="error,evm=debug,sc_rpc_server=info,runtime::revive=debug" ./target/release/revive-dev-node --dev --tmp
RUST_LOG="info,eth-rpc=debug" ./target/release/eth-rpc --dev
```

知识点：

- `--dev` 表示开发模式
- `--tmp` 表示使用临时链数据
- `RUST_LOG` 用于控制日志级别



## 5. Lock.sol 合约知识点

### 5.1 Solidity 版本

```solidity
pragma solidity ^0.8.28;
```

表示该合约使用 Solidity `0.8.28` 及兼容版本编译。

### 5.2 状态变量

```solidity
uint public unlockTime;
address payable public owner;
```

知识点：

- `uint` 是无符号整数
- `public` 会自动生成 getter 函数
- `address payable` 表示该地址可以接收原生币转账

### 5.3 事件

```solidity
event Withdrawal(uint amount, uint when);
```

事件用于记录链上操作日志，方便前端、脚本或区块浏览器追踪合约行为。

### 5.4 payable 构造函数

```solidity
constructor(uint _unlockTime) payable {
    require(
        block.timestamp < _unlockTime,
        "Unlock time should be in the future"
    );

    unlockTime = _unlockTime;
    owner = payable(msg.sender);
}
```

知识点：

- `constructor` 在合约部署时执行一次
- `payable` 表示部署时可以向合约发送原生币
- `require` 用于条件检查，不满足条件时 revert
- `block.timestamp` 表示当前区块时间
- `msg.sender` 表示调用者地址，在构造函数中就是部署者

### 5.5 代码与测试一致性

当前 `Lock.sol` 中只有：

```solidity
function withdrawTest() public payable {}
```

但测试文件中调用的是：

```ts
lock.withdraw()
```

这说明合约代码和测试代码不一致。实际开发中，测试必须和合约接口保持同步，否则测试会失败。

## 6. Hardhat 配置知识点

相关文件：

- `hardhat-example/hardhat.config.ts`

### 6.1 Solidity 编译器配置

```ts
solidity: "0.8.28"
```

用于指定 Hardhat 编译 Solidity 合约时使用的编译器版本。

### 6.2 Polkadot Hub Testnet 网络配置

```ts
networks: {
  polkadotTestnet: {
    url: "https://services.polkadothub-rpc.com/testnet",
    chainId: 420420417,
    accounts: [vars.get("PRIVATE_KEY")],
  },
}
```

知识点：

- `url` 是 RPC 地址
- `chainId` 用于标识目标链
- `accounts` 配置部署或交易使用的钱包私钥
- `vars.get("PRIVATE_KEY")` 从 Hardhat vars 中读取私钥

### 6.3 合约验证配置

```ts
etherscan: {
  apiKey: {
    polkadotTestnet: "no-api-key-needed",
  },
  customChains: [
    {
      network: "polkadotTestnet",
      chainId: 420420417,
      urls: {
        apiURL: "https://blockscout-testnet.polkadot.io/api",
        browserURL: "https://blockscout-testnet.polkadot.io/",
      },
    },
  ],
}
```

这里配置了 Blockscout 的 API 和浏览器地址，使 Hardhat 可以在 Polkadot Hub Testnet 上验证合约源码。

## 7. Hardhat 测试知识点

相关文件：

- `hardhat-example/test/Lock.ts`

### 7.1 测试结构

```ts
describe("Lock", function () {
  it("Should set the right unlockTime", async function () {
    ...
  });
});
```

Hardhat 测试通常使用 Mocha + Chai。

常见结构：

- `describe`：测试分组
- `it`：具体测试用例
- `expect`：断言

### 7.2 Fixture

```ts
const { lock, unlockTime } = await loadFixture(deployOneYearLockFixture);
```

`loadFixture()` 会先执行一次部署逻辑，然后通过快照复用链状态，提高测试效率。

### 7.3 时间控制

```ts
const unlockTime = (await time.latest()) + ONE_YEAR_IN_SECS;
await time.increaseTo(unlockTime);
```

Hardhat Network 可以模拟链上时间变化，适合测试时间锁、质押、解锁等逻辑。

### 7.4 revert 测试

```ts
await expect(Lock.deploy(latestTime, { value: 1 })).to.be.revertedWith(
  "Unlock time should be in the future"
);
```

用于验证某些非法操作是否会按预期失败。

### 7.5 事件测试

```ts
await expect(lock.withdraw())
  .to.emit(lock, "Withdrawal")
  .withArgs(lockedAmount, anyValue);
```

用于验证交易是否触发指定事件，以及事件参数是否正确。

### 7.6 余额变化测试

```ts
await expect(lock.withdraw()).to.changeEtherBalances(
  [owner, lock],
  [lockedAmount, -lockedAmount]
);
```

用于验证转账或提现是否导致账户余额按预期变化。

## 8. Hardhat Ignition 部署知识点

相关文件：

- `hardhat-example/ignition/modules/Lock.ts`

### 8.1 buildModule

```ts
const LockModule = buildModule("LockModule", (m) => {
  ...
});
```

Hardhat Ignition 使用模块化方式描述部署流程。

### 8.2 部署参数

```ts
const unlockTime = m.getParameter("unlockTime", JAN_1ST_2030);
const lockedAmount = m.getParameter("lockedAmount", ONE_GWEI);
```

`m.getParameter()` 用于定义可配置部署参数，并提供默认值。

### 8.3 部署合约

```ts
const lock = m.contract("Lock", [unlockTime], {
  value: lockedAmount,
});
```

这里表示：

- 部署 `Lock` 合约
- 构造函数参数是 `unlockTime`
- 部署时向合约发送 `lockedAmount`

## 9. hardhat-pvm-example：Hardhat Polkadot/PVM 工程

相关文件：

- `hardhat-pvm-example/hardhat.config.ts`
- `hardhat-pvm-example/contracts/MyToken.sol`
- `hardhat-pvm-example/test/MyToken.ts`
- `hardhat-pvm-example/ignition/modules/MyToken.ts`

### 9.1 @parity/hardhat-polkadot

```ts
import "@parity/hardhat-polkadot";
```

这个插件让 Hardhat 能够支持 Polkadot 智能合约开发环境。

### 9.2 EVM 与 PVM target

```ts
hardhat: {
    polkadot: {
        target: "evm",
    },
}
```

```ts
polkadotHubTestnet: {
    polkadot: {
        target: "pvm",
    },
}
```

知识点：

- `target: "evm"` 表示面向 EVM 兼容目标
- `target: "pvm"` 表示面向 Polkadot Virtual Machine

### 9.3 本地节点配置

```ts
nodeConfig: {
    nodeBinaryPath: "./bin/dev-node",
    rpcPort: 8000,
    dev: true,
}
```

用于指定本地开发节点二进制文件和 RPC 端口。

### 9.4 适配器配置

```ts
adapterConfig: {
    adapterBinaryPath: "./bin/eth-rpc",
    dev: true,
}
```

`eth-rpc` 适配器让 EVM 工具能够通过 Ethereum 风格 RPC 与 Polkadot 环境交互。

### 9.5 环境变量

```ts
import dotenv from "dotenv";
dotenv.config();
```

```ts
accounts: [process.env.PRIVATE_KEY!]
```

这里使用 `.env` 文件读取私钥，比直接写死在代码里更安全。

## 10. MyToken.sol：ERC20 Token 合约知识点

相关文件：

- `hardhat-pvm-example/contracts/MyToken.sol`

### 10.1 OpenZeppelin 合约库

```solidity
import "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import "@openzeppelin/contracts/token/ERC20/extensions/ERC20Burnable.sol";
import "@openzeppelin/contracts/utils/Pausable.sol";
import "@openzeppelin/contracts/access/AccessControl.sol";
```

OpenZeppelin 是常用的智能合约基础库，可以避免重复实现标准逻辑。

这里使用了：

- `ERC20`：标准 Token 功能
- `ERC20Burnable`：允许销毁 Token
- `Pausable`：允许暂停合约操作
- `AccessControl`：基于角色的权限控制

### 10.2 角色定义

```solidity
bytes32 public constant PAUSER_ROLE = keccak256("PAUSER_ROLE");
bytes32 public constant MINTER_ROLE = keccak256("MINTER_ROLE");
```

角色使用 `bytes32` 标识，通常通过 `keccak256()` 生成。

### 10.3 构造函数

```solidity
constructor(uint256 initialSupply) ERC20("MyToken", "MTK") {
    _grantRole(DEFAULT_ADMIN_ROLE, msg.sender);
    _grantRole(PAUSER_ROLE, msg.sender);
    _grantRole(MINTER_ROLE, msg.sender);

    _mint(msg.sender, initialSupply);
}
```

部署时完成：

- 设置 Token 名称：`MyToken`
- 设置 Token 符号：`MTK`
- 给部署者管理员角色
- 给部署者暂停权限
- 给部署者铸币权限
- 给部署者铸造初始供应量

### 10.4 pause / unpause

```solidity
function pause() public onlyRole(PAUSER_ROLE) {
    _pause();
}

function unpause() public onlyRole(PAUSER_ROLE) {
    _unpause();
}
```

只有拥有 `PAUSER_ROLE` 的账户可以暂停或恢复合约。

### 10.5 mint

```solidity
function mint(address to, uint256 amount) public onlyRole(MINTER_ROLE) {
    _mint(to, amount);
}
```

只有拥有 `MINTER_ROLE` 的账户可以铸造新 Token。

### 10.6 重写 _update

```solidity
function _update(
    address from,
    address to,
    uint256 amount
) internal override whenNotPaused {
    super._update(from, to, amount);
}
```

OpenZeppelin ERC20 在转账、铸造、销毁时都会经过 `_update()`。

这里重写 `_update()` 并加上 `whenNotPaused`，表示暂停状态下禁止 Token 状态更新。

## 11. MyToken 测试知识点

相关文件：

- `hardhat-pvm-example/test/MyToken.ts`

### 11.1 beforeEach

```ts
beforeEach(async () => {
    [owner, addr1, addr2] = await hre.ethers.getSigners();
    const MyToken = await hre.ethers.getContractFactory("MyToken");
    token = await MyToken.deploy(toWei("1000000"));
    await token.waitForDeployment();
});
```

每个测试用例执行前都会重新部署一个新合约，保证测试之间互不影响。

### 11.2 ERC20 精度处理

```ts
const toWei = (value: string) => hre.ethers.parseUnits(value, 18);
```

ERC20 通常使用 18 位小数。`parseUnits(value, 18)` 用于把人类可读的 Token 数量转换成链上整数。

### 11.3 测试初始供应量

```ts
expect(balance).to.equal(toWei("1000000"));
```

验证部署者是否拿到了初始 Token。

### 11.4 测试权限控制

```ts
await expect(token.connect(addr1).mint(addr2.address, amount))
    .to.be.revertedWithCustomError(token, "AccessControlUnauthorizedAccount");
```

验证没有 `MINTER_ROLE` 的账户不能铸币。

### 11.5 测试暂停逻辑

```ts
await token.pause();
await expect(token.transfer(addr1.address, 1)).to.be.revertedWithCustomError(
    token,
    "EnforcedPause",
);
```

验证暂停后转账会失败。

## 12. Foundry 工程知识点

相关文件：

- `my-foundry-project/src/Counter.sol`
- `my-foundry-project/test/Counter.t.sol`
- `my-foundry-project/script/Counter.s.sol`
- `my-foundry-project/foundry.toml`
- `my-foundry-project/README.md`

### 12.1 Foundry 组成

Foundry 是 Rust 编写的以太坊开发工具链，主要包括：

- `Forge`：编译、测试、部署
- `Cast`：命令行链上交互工具
- `Anvil`：本地测试节点
- `Chisel`：Solidity REPL

### 12.2 Counter 合约

```solidity
contract Counter {
    uint256 public number;

    function setNumber(uint256 newNumber) public {
        number = newNumber;
    }

    function increment() public {
        number++;
    }

    function getNumber() public view returns (uint256) {
        return number + 135;
    }
}
```

知识点：

- `number` 是链上状态变量
- `setNumber()` 修改状态
- `increment()` 修改状态
- `getNumber()` 是只读函数
- `public` 状态变量会自动生成 getter

### 12.3 Foundry 测试

```solidity
function setUp() public {
    counter = new Counter();
    counter.setNumber(0);
}
```

`setUp()` 会在每个测试用例前执行。

```solidity
function test_Increment() public {
    counter.increment();
    assertEq(counter.number(), 1);
}
```

普通单元测试验证 `increment()` 是否正确。

```solidity
function testFuzz_SetNumber(uint256 x) public {
    counter.setNumber(x);
    assertEq(counter.number(), x);
}
```

`testFuzz_` 开头的测试是 fuzz 测试。Foundry 会自动生成多组输入值，测试函数在不同输入下是否都成立。

### 12.4 Foundry 部署脚本

```solidity
uint256 deployerPrivateKey = vm.envUint("PRIVATE_KEY");

vm.startBroadcast(deployerPrivateKey);

counter = new Counter();

vm.stopBroadcast();
```

知识点：

- `vm.envUint()` 从环境变量读取私钥
- `vm.startBroadcast()` 开始广播真实交易
- `new Counter()` 部署合约
- `vm.stopBroadcast()` 停止广播

### 12.5 foundry.toml

```toml
[profile.default]
src = "src"
out = "out"
libs = ["lib"]
solc_version = "0.8.28"
```

用于配置 Foundry 项目：

- 源码目录
- 编译输出目录
- 依赖目录
- Solidity 编译器版本

### 12.6 Polkadot Testnet 验证配置

```toml
[etherscan]
polkadot-testnet = { key = "verifyContract", url = "https://api.routescan.io/v2/network/testnet/evm/420420417/etherscan" }
```

用于配置 Foundry 的合约验证服务。

## 13. 常用命令总结

### 13.1 ethers 本地脚本

```bash
npm run start
```

运行 TypeScript 脚本，连接 RPC，查询余额并发送转账。

### 13.2 Hardhat

```bash
npx hardhat compile
npx hardhat test
npx hardhat node
```

分别用于编译、测试和启动本地节点。

### 13.3 Hardhat Ignition 部署

```bash
npx hardhat ignition deploy ./ignition/modules/Lock.ts
```

部署 Hardhat Ignition 模块。

部署到 Polkadot Hub Testnet：

```bash
npx hardhat ignition deploy ./ignition/modules/Lock.ts --network polkadotTestnet
```

### 13.4 Hardhat 合约验证

```bash
npx hardhat verify --network polkadotTestnet <contract-address>
```

如果构造函数有参数，需要在地址后面补充构造参数。

### 13.5 Foundry

```bash
forge build
forge test
forge fmt
forge snapshot
anvil
```

含义：

- `forge build`：编译
- `forge test`：测试
- `forge fmt`：格式化
- `forge snapshot`：生成 gas 快照
- `anvil`：启动本地节点

### 13.6 Foundry 部署到 Polkadot Testnet

```bash
forge create src/Counter.sol:Counter \
    --chain polkadot-testnet \
    --rpc-url https://services.polkadothub-rpc.com/testnet \
    --private-key $PRIVATE_KEY \
    --broadcast
```

### 13.7 Foundry 脚本部署

```bash
forge script script/Counter.s.sol:CounterScript \
    --chain polkadot-testnet \
    --broadcast
```

### 13.8 Foundry 合约验证

```bash
forge verify-contract <contract-address> \
    src/Counter.sol:Counter \
    --chain polkadot-testnet
```

## 14. 重点概念对照

### 14.1 Provider 与 Wallet

- `Provider` 负责连接节点和读取链上数据
- `Wallet` 负责管理私钥和签名交易

### 14.2 读操作与写操作

- 读操作：不修改链上状态，通常不需要签名，不产生交易
- 写操作：修改链上状态，需要签名，需要 gas，需要等待确认

### 14.3 ABI 与 bytecode

- ABI：合约接口说明，用于链下调用和结果解码
- bytecode：合约编译后的 EVM/PVM 可执行代码，用于部署

### 14.4 wei、ETH、Token decimals

- 原生币底层单位是 wei
- `parseEther()`：ETH 转 wei
- `formatEther()`：wei 转 ETH
- ERC20 常用 `parseUnits(value, decimals)` 处理精度

### 14.5 权限控制

`MyToken` 使用 `AccessControl` 实现角色权限：

- `DEFAULT_ADMIN_ROLE`：管理员
- `PAUSER_ROLE`：暂停权限
- `MINTER_ROLE`：铸币权限

### 14.6 暂停机制

`Pausable` 可以在紧急情况下暂停合约核心操作。`MyToken` 通过重写 `_update()` 实现暂停转账、铸造和销毁。

### 14.7 测试工具差异

Hardhat 测试特点：

- 使用 TypeScript
- 使用 Mocha + Chai
- 适合和前端/脚本开发结合

Foundry 测试特点：

- 使用 Solidity 编写测试
- 执行速度快
- 原生支持 fuzz test
- 适合偏底层和合约密集型测试

## 15. 容易出错的地方

### 15.1 私钥硬编码

`local-dev/src/index.ts` 中直接写了私钥。这只适合本地测试，真实项目应使用环境变量或密钥管理工具。

### 15.2 测试和合约不一致

`hardhat-example/test/Lock.ts` 测试调用了 `withdraw()`，但当前 `Lock.sol` 里没有该函数。需要修改合约或测试，保持接口一致。

### 15.3 网络名称不一致

Hardhat 命令里的 `--network` 名称必须和 `hardhat.config.ts` 中配置的网络名称一致。

### 15.4 单位转换错误

链上金额不能直接用小数或普通字符串随意传入。原生币应使用 `parseEther()`，ERC20 应使用 `parseUnits()`。

### 15.5 没有等待交易确认

发送交易后要使用 `tx.wait()` 或相关等待方法确认上链，否则后续读取可能还是旧状态。

## 16. 本节课应该掌握的能力

学完 `lesson-3` 后，应该能做到：

1. 使用 `ethers.js` 连接 RPC 节点
2. 使用私钥创建钱包并发送交易
3. 区分 Provider 和 Wallet 的职责
4. 使用 Hardhat 编译、测试、部署 Solidity 合约
5. 配置 Polkadot Hub Testnet 网络
6. 使用 Hardhat Ignition 管理部署流程
7. 使用 OpenZeppelin 编写 ERC20 Token
8. 使用 AccessControl 做权限控制
9. 使用 Pausable 实现暂停机制
10. 使用 Hardhat 编写 TypeScript 测试
11. 使用 Foundry 编写 Solidity 测试
12. 使用 Foundry 脚本部署合约
13. 理解 EVM、PVM、Polkadot Revive 和 eth-rpc 的关系
14. 对合约进行源码验证

## 17. 一句话总结

`course/lesson-3` 的核心，是通过多个示例工程展示智能合约开发的完整流程：本地链下交互、Hardhat 开发测试部署、Polkadot PVM 适配、OpenZeppelin 合约复用，以及 Foundry 工具链的使用。
