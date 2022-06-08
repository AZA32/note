# 以太坊

### MPT

由压缩前缀树 Patricia Tree + 默克尔树 Merkle Tree 组成

#### ![img](http://cdn.blocketh.top/img/4.png)

#### 场景

* 交易树
* 收据树

### 区块

* Head
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
	Number      *big.Int       `json:"number"           
	GasLimit    uint64         `json:"gasLimit"         //当前每个区块的 gas 使用限制值
	GasUsed     uint64         `json:"gasUsed"          //该区块中用于交易的 gas 消耗值
	Time        uint64         `json:"timestamp"        //时间戳
	Extra       []byte         `json:"extraData"        //与该区块相关的 32 字节数据
	MixDigest   common.Hash    `j
	// BaseFee was added by EIP-1559 and is ignored in legacy headers.
	BaseFee *big.Int `json:"baseFeePerGas" rlp:"optional"`
}
```

**Bloom Filter**

当前区块所有交易中 Bloom Filter 的并集，用来判断交易类型是否存在，Solidity 中 Event 事件

#### Receipts

### StateRoot

StateRoot 数据结构是 Merkle Patric Trie(MPT)，叶子节点存储以太坊账户，Key 为以太坊地址（哈希值），value 为账户对象（RLP编码序列化）

#### 账户

**外部账户** (Externally Owned Account, **EOA** ) 与 **智能合约** (Contract Account, **CA** )。

*   `nonce`：已执行交易总数，用来标示该账户发出的交易数量；

    > 对重放攻击有防护作用
* `balance`： 持币数量，记录用户的以太币余额；
* `storage hash`： 存储区的哈希值，指向智能合约账户的存储数据区；
* `code hash` ：代码区的哈希值，指向智能合约账户存储的智能合约代码（存储区的内容通过散列函数得出校验哈希值）。

| 项              | 外部账户 | 合约账户       |
| -------------- | ---- | ---------- |
| 私钥 private Key | ✔️   | ✖️         |
| 余额 balance     | ✔️   | ✔️         |
| 代码 code        | ✖️   | ✔️         |
| 多重签名           | ✖️   | ✔️         |
| 控制方式           | 私钥控制 | 通过外部账户执行合约 |

![image-20220402142428655](http://cdn.blocketh.top/img/image-20220402142428655.png)

`stateRoot`叶子节点是账户信息(addr => account)

![image-20220402142409642](http://cdn.blocketh.top/img/image-20220402142409642.png)

账户状态哈希值 `StateRoot`，是合约所拥有的方法、字段信息构成的一颗默克尔压缩前缀树，合约状态中的任意一项细微变动都最终引起 `StateRoot` 变化，因此合约状态变化会反映在账户的`StateRoot`上

### 共识机制（GHOST）

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

#### ERC20

**问题**：

* 转账无法携带额外的信息
* 没有转账回调
  * 依赖`approve`
  * 误转合约被锁死

#### ERC777

* `send(dest,vale,data)`：data（额外的信息）
* 通过全局注册表（ERC1820）注册监听回调
  * EOA也可以实现回调

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

#### EIP2612

链下签名

* 链下进行签名，签名信息在执行转账交易时提交到链上，授权和转账在一笔交易里完成
* 转账可以交给第三方来提交，避免ERC20交易需要ETH来提供手续费

### 共识机制

**POW**

### EVM

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
