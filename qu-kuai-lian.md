# 区块链

### 介绍

区块链是由区块组成的单向列表（和普通链表的区别是hash指针代表了普通指针），每个区块记录了一系列交易，并且指向上一个区块。区块链的本质是一个去中心化的点对点分布式数据库，也是一个分布式计算机（提供运行脚本、只能合约的能力）

**防篡改**

#### Block

* Data
* Hash
* Prev\_Hash

### 密码学

#### 哈希算法

哈希算法又被称为散列函数，它可以将任意长度的输入转化为固定长度的输出。Hash算法具有以下特征

* 确定性：任意长度的输入值通过哈希函数得到的哈希值长度都是一定的，例如Hash256算法，它的哈希值都是256位字节的字符长度。
* 不可逆性：X -> H(X)，H(X) 无法反推出 X
* 抗碰撞能力：任何两个不同的输入值得到的哈希值都不相同，稍微改变输入值的某个参数，得到的哈希值都天差地别

**应用**

*   BTC 工作量证明

    <img src="http://cdn.blocketh.top/img/8974afd1dbf2041811bf4b43a28afa84aabf8fa2.png@759w_306h_progressive.webp" alt="img" data-size="original">
*   BTC-默克尔树

    <img src="http://cdn.blocketh.top/img/fed9506288e59bf4dbc6072e1bec70d7.png" alt="图片" data-size="original">

#### 非对称加密

包含 **公钥** 和 **私钥**，如果用公钥对数据加密，那么只能用对应的私钥解密。如果用私钥对数据加密，只能用对应的公钥进行解密。因为加密和解密用的是不同的密钥，所以称为非对称加密。

**应用**

* 账户

**常用算法**

* RSA
* DSA（数字签名算法）：DSA 仅能用于数字签名，不能进行数据加密解密，其安全性和 RSA 相当，但其性能要比 RSA 快。
* ECDSA（椭圆曲线签名算法）：ECC（椭圆曲线密码学）和 DSA 的结合，ECC 可以使用更小的秘钥，更高的效率，提供更高的安全保障

**应用**

* 账户

#### 签名

发送方用自己的私钥对发送信息进行所谓的加密运算，得到一个hash值，该hash值就是签名。使用时需要将签名和信息发给接收方。接受者用发送方公开的公钥和接收到的信息对签名及逆行验证，通过认证，说明接受到的信息是完整的，准确的，否则说明消息来源不对

**应用**

* 交易

### 共识机制

区块链本质是一个分布式系统，如何保障多个节点之间的数据一致性

#### POW

### 分叉

#### 硬分叉

把原有的一条链直接分成了两条完全不同的链，一条是旧链，一条是新链，旧链就是不愿意让币种分叉的社区成员所坚持的原有的链，新链就是社区成员希望在原有区块链上进行技术优化改进所生成的链，**硬分叉发生后，各自产生新的区块，数据不再同步。**

**场景**

* BTC和BCH
* ETH和ETC

#### 软分叉

一种临时性的场景，在一条链上，软分叉会进行一些升级，但是不会影响整个系统的稳定性和有效性，旧节点会兼容新节点，只是新节点不兼容旧节点而已，二者依然可以共存在一条链上。

### 攻击

#### 双花攻击（发送者不诚实）

一笔钱花费两次

攻击者在区块为N的高度上发起交易，同时沿着N-1的区块高度再创建出一条隐藏的分叉链并且排除自己的交易。由于区块链遵循最长链原则，只要攻击者的算力大于全网51%即可完成攻击。

#### 女巫攻击

恶意节点创建多个虚假身份，可以以多数票击败网络上真实的节点，可以拒绝接收或者传输区块，从而有效的防止其它用户进入网络。

在大规模中女巫攻击中，可以轻易更改交易的顺序，防止交易被确认，甚至可以逆转交易，导致双重支付等问题。一般产生于使用投票算法的共识机制中。

#### 重放攻击（接收者不诚实）

重放攻击是发生在区块链硬分叉之时的一种独特现象。进行硬分叉之后，区块链发生永久性分歧并产生两条历史交易、地址、私钥以及余额等完全对应的链，导致在其中一条链上的交易在另一条链上很可能是完全合法的。你在其中一条链上发起的交易，可以到另一条链上去重新广播，也可能会得到确认，这就是“重放攻击”。

**防止**

重放攻击需要监听一条链上的交易情况并转发至另一分叉链上以窃取资金，Nonce的使用可以在一定程度上避免分叉情况下的重放攻击。如果在一条分叉链上发起的交易，其使用的Nonce已被另一条分叉链中的相同用户发起的交易所使用，那么节点将不会广播该交易

**场景**

* ETH ETC

### 零知识证明

零知识证明是指一方（证明者）向另一方（验证者）证明一个陈述是正确的，而无需透露除该陈述是正确的外的任何信息。

### 钱包

#### HD钱包

**Bip32（为了避免管理一堆私钥提出的分层推导方案）**

定义了HD wallet，是一个系统可以从单一`seed`产生一树状结构储存多组`keypairs`（私钥和公钥）。这样只需保存一个种子即可，私钥可以推到出来。可以方便的备份、转移到其他相容装置（因为都需要`seed`）,以及分层的权限控制。

![image-20220402142337314](http://cdn.blocketh.top/img/image-20220402142337314.png)

**Bip39**

将`speed`用方便记压和书写的单词表示，一般由12个单词组成，称为mnemonic code（助记词）

**Bip44**

基于Bip32，赋予树状结构层次中的各层特殊的意义，让`speed`支持多链、多账户

```
m / purpose' / coin_type' / account' / change / address_index
//purporse': 固定值44', 代表是BIP44
//coin_type': 这个代表的是币种, 可以兼容很多种币, 比如BTC是0', ETH是60'
//btc一般是 m/44'/0'/0'/0
//eth一般是 m/44'/60'/0'/0
```