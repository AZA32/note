# 以太坊

### 以太坊基本原理

**平均出块时间**：12s

**共识机制**：Ghost

### MPT

由压缩前缀树 Patricia Tree + 默克尔树 Merkle Tree 组成

#### ![img](http://cdn.blocketh.top/img/4.png)

#### 场景

* 交易树
* 收据树

### 区块

* Head
* Uncle
* Transactions

#### 区块头

```go
type Header struct {
	ParentHash  common.Hash    `json:"parentHash"       //父块keccak哈希
	UncleHash   common.Hash    `json:"sha3Uncles"       //叔块keccak哈希
	Coinbase    common.Address `json:"miner"            //
	Root        common.Hash    `json:"stateRoot"        //世界状态树的根哈希值
	TxHash      common.Hash    `json:"transactionsRoot" //交易树的根哈希值
	ReceiptHash common.Hash    `json:"receiptsRoot"     //收据树的根哈希值
	Bloom       Bloom          `json:"logsBloom"        //事件地址和事件 topic 的布隆滤波器
	Difficulty  *big.Int       `json:"difficulty"       //前一个区块的难度
	Number      *big.Int       `json:"number"           //区块高度
	GasLimit    uint64         `json:"gasLimit"         //当前每个区块的 gas 使用限制值
	GasUsed     uint64         `json:"gasUsed"          //该区块中用于交易的 gas 消耗值
	Time        uint64         `json:"timestamp"        //时间戳
	Extra       []byte         `json:"extraData"        //与该区块相关的 32 字节数据
	MixDigest   common.Hash    `j
	// BaseFee was added by EIP-1559 and is ignored in legacy headers.
	BaseFee *big.Int `json:"baseFeePerGas" rlp:"optional"`
}
```

**stateRoot**

stateRoot 结构为MPT，叶子节点存储以太坊账户(addr => account)，Key 为以太坊地址（哈希值），value 为账户对象（RLP编码序列化）

![image-20220402142428655](http://cdn.blocketh.top/img/image-20220402142428655.png)

**账户**

| 项              | 外部账户 | 合约账户       |
| -------------- | ---- | ---------- |
| 私钥 private Key | ✔️   | ✖️         |
| 余额 balance     | ✔️   | ✔️         |
| 代码 code        | ✖️   | ✔️         |
| 多重签名           | ✖️   | ✔️         |
| 控制方式           | 私钥控制 | 通过外部账户执行合约 |

**外部账户（EOA）**

*   **nonce**：已执行交易总数，用来标示该账户发出的交易数量

    > 对重放攻击有防护作用
* **balance**：以太币余额

**合约账户**

* **nonce**
* **balance**
*   **storageRoot**： 另一个MPT，存储区的哈希值，指向智能合约账户的存储数据区；

    > 合约状态中的任意一项细微变动都最终引起**stateRoot**的变化
* **codeHash** ：代码哈希值，指向智能合约账户存储的智能合约代码（存储区的内容通过散列函数得出校验哈希值）

**Bloom Filter**

当前区块所有交易（每个交易完成后会产生一个收据，收据中会包含一个Bloom Filter，记录这个交易的类型、地址等其他信息）中 Bloom Filter 的并集，用来判断交易类型是否存在，Solidity 中 Event 事件

> 当查找某段时间某个智能合约相关的所有交易:
>
> 在区块头中查找是否存在相关的交易类型
>
> * 存在：则在区块内部的所有收据里的Bloom Filter中查找。
> * 不存在：直接查找下个区块。

**Transactions（交易树）**

每发布一个区块，区块中的交易会形成一颗 Merkle Tree，即交易树

**Receipts（收据树）**

每个交易执行完毕后，都会有一个收据，这个收据记录交易的相关信息。每个区块中，所有交易的收据会组织成一颗收据树，与交易树是一一对应的，同样也是 MPT。

**作用**

在以太坊中最重要的功能是加入了智能合约，而智能合约的执行过程比较复杂，收据树的作用是利于系统快速查询执行结果。

#### Transaction

* **To**：接收方地址
* **Value**：交易方发送的Ether数量（wei）
* **GasPrice**：交易费支付的Gas单价
* **Data**：可变长的二进制数据载荷
*   **R、S、V**

    > 函数选择器 keccak256(function\_name(para\_type))

![image-20220402142409642](http://cdn.blocketh.top/img/image-20220402142409642.png)

### 共识机制

#### POW

* 防止ASIC算力中心化
* DAG + Hashimoto：DAG用来构造伪随机数据集，Hashimoto利用数据集计算目标hash
* 有利于轻节点验证区块
* 有利于GPU

![image-20220611144651437](http://cdn.blocketh.top/img/image-20220611144651437.png)

**DAG**

*   Epoch = 30000 Blocks

    每一个世代，整个DAG重新生成
*   每个世代种子之间互相依赖

    Speed Hash\[0] = KEC(32 Bytes 0)

    Speed Hash\[i] = KEC(Speed Hash\[i-1])
*   种子生成CACHE

    Cache init = 16 MB

    Cache Growth = 128 K

    Cache Size = 64 Bytes的素数倍

    每个Cache的单位是64B
*   CACHE生成DATA

    Data init = 1 GB

    Data Growth = 8 M 每世代

    Data Size = 128Bytes的素数倍

    每个Data单位是MixBytes是128 Bytes

**Hashimoto**

**难度计算**

计算一个区块的难度时，需要以下输入：

* parent\_timestamp：上一个区块产生的时间
* parent\_diff：上一个区块的难度
* block\_timestamp：当前区块产生的时间
* block\_number：当前区块的序号

$$
diff = parent_diff + 难度调整 + 难度炸弹
$$

$$
难度调整 = parent_diff / 2048 * max((2 if len(parent.uncles) else 1) - ((block_timestamp - parent_timestamp) // 9), -99)))
$$

$$
难度炸弹 = INT(2**((block_number // 100000) - 2))
$$

#### POA

主要用于测试网络，测试网消耗POW算力成本太高

存在两种节点类型：

* 认证节点
* 非认证节点

### EVM

* 基于栈的虚拟机，一次调用产生一个实例
* 每一个栈的元素大小是32字节
* 大小无限制，调用深度上限1024
* 临时、永久存储

#### 指令集

#### 代码

#### 全局状态

* Account States
* Balances
* Storage and Code

#### 执行环境

* Code owner
* sender
* Input data
* value
* Machine code
* Block header

#### 虚拟机实例状态

* Gas available
* Program counter
* Memory contents
* Stack contents

#### 定价策略

#### 预编译合约

### EIP

EIP(Ethereum Improvement Proposals)：以太坊改进提案

* Standard Track EIP
  * Core（核心协议）
  * Networking（网络协议）
  * Interface（接口协议）
  *   ERC

      Ethereum Request For Comment：以太坊意见征求稿，用于记录以太坊上应用级的各种开发标准和协议。
* Meta EIP
* informational EIP（信息类协议）

#### EIP-55

地址大小写无关。校验和检测准确99.856%

1. Hash = Keccak256(Addr)
2. Hash = MSB(Hash) 20 Byte
3. Hash和原有地址进行比对，Hash中大于8的十六进制数对应源地址位改为大写，把校验和嵌入到地址本身

#### EIP712

签名信息格式化展示

```solidity
bytes32 eip712DomainHash = keccak256(
    abi.encode(
        keccak256(
            "EIP712Domain(string name,string version,uint256 chainId,address verifyingContract)"
        ),
        keccak256(bytes(name())), // ERC-20 Name
        keccak256(bytes("1")),    // Version
        chainid(),
        address(this)
    )
);
```

这样可以确保仅在正确的链ID上将签名用于我们给定的通证合约地址。chainID是在以太坊经典分叉之后引入（以太坊经典network id 依旧为 1）， 用来精确识别在哪一个网络。 可以在此处查看现有[chain ID的列表](https://medium.com/@piyopiyo/list-of-ethereums-major-network-and-chain-ids-2bc58e928508)。

#### EIP1882

**通用可升级代理标准(UUPS)**

#### EIP1884

EIP1884 最具争议的变化是`SLOAD`从 200 到 800 的 gas 成本增加

`transfer`、`send`最初是作为一种减轻[重入漏洞](https://consensys.github.io/smart-contract-best-practices/known\_attacks/#reentrancy)的简单方法而创建的[ 10](https://consensys.github.io/smart-contract-best-practices/known\_attacks/#reentrancy). 虽然他们成功地实现了这一目的，但他们的方法过于严厉，导致了诸如此类的问题。以太坊安全社区的共识是不鼓励使用它们，而是使用[检查-效果-交互来代替 16](https://solidity.readthedocs.io/en/v0.5.11/security-considerations.html#use-the-checks-effects-interactions-pattern)模式或自定义解决方案，例如我们的[ReentrancyGuard 9](https://docs.openzeppelin.com/contracts/2.x/api/utils#ReentrancyGuard)合同。

#### EIP1967

对委托调用代理所使用的插槽进行了标准化。这允许诸如[Etherscan](https://medium.com/etherscan-blog/and-finally-proxy-contract-support-on-etherscan-693e3da0714b)浏览器能轻松识别这些代理(因为在该特定插槽中具有类似地址值的任何合约很可能是代理)并解析出对应的合约的地址。

遵循EIP 1967实现随机存储示例：

```
bytes32 private constant implementationPosition = bytes32(uint256(
  keccak256('eip1967.proxy.implementation')) - 1
));
```

#### EIP2612

链下签名

* 链下进行签名，签名信息在执行转账交易时提交到链上，授权和转账在一笔交易里完成
* 转账可以交给第三方来提交，避免ERC20交易需要ETH来提供手续费

### ERC

#### ERC20

**问题**：

* 转账无法携带额外的信息
* 没有转账回调
  * 依赖`approve`
  * 误转合约被锁死

#### ERC721

*   所有权查询

    ```solidity
    ownerOf(uint256 tokenId)
    ```
*   对应账户余额

    ```solidity
    balanceOf(adress tokenOwner)
    ```
*   许可份额

    ```solidity
    approve(address approved,uint256 tokenId)
    transferFrom(address from,address to,uint256 tokenId)
    ```

    `safetransFrom`会查询`to`地址和`tokenId`的有效性，如果`to`地址是合约地址还会触发`onERC721Received`函数
* 许可相关查询
  *   查询token对应的被许可地址

      ```solidity
      getApproved(uint256 _tokenId)
      ```
  *   指定或撤销许可权限

      ```solidity
      setApprovalForAll(address _operator,bool _approved)
      ```
  *   查询许可权限

      ```solidity
      isApprovedForAll(address _owner,address _operator)
      ```

#### ERC165

在合约中增加一个标准，检测是否支持某一个方法

#### ERC777

* `send(dest,vale,data)`：data（额外的信息）
* 通过全局注册表（ERC1820）注册监听回调
  * EOA也可以实现回调

***

### EV M

代码组织结构

* core
  * state\_process.go：EVM入口函数
  * state\_transition.go：调用EVM前的准备
* core/vm
  * analysis.go：实现了合约代码扫描检测的工具函数
  * common.go：实现了一些公用的辅助函数
  * contract.go：实现了`contract`这个重要的结构和方法
  * contracts.go：实现了主要预编译合约
  * evm.go：实现了evm虚拟机的结构定义和相关的重要方法
  * gas\_table.go：实现了各类不同指令所需要的动态计算gasg的方法
  * instructions.go：实现了各类指令的实际执行代码
  * interface.go：定义了StatDB接口和Callcontext接口
  * interpreter.go：实现了Opcode解释器
  * jump\_table.go：非常重要的lookup table，针对每一个具体的opCode。分别指向gas\_table和instructions的具体实现。解释器直接会根据opCode来进行查找
  * memory.go：实现了evm的临时memory存储
  * opcodes.go：定义了所有的opCode以及其和字符串的映射表
  * stack.go：evm中所用到栈的实现。
* params/
  * protocol\_params.go：定义了各个不同代际的evm各类操作的gas费用标准。
