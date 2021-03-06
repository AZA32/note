# 智能合约

### Solidity

​ solidity是一种静态类型、面向合约的语言

#### 变量

**数据类型**

**值类型**

*   整数

    ​ delete整数 = 0
*   bool

    ​ 1用来表示 true，0表示 false
*   Address

    * balance
    * transfer
    * send
    * delegatecall

    > 支持运算符：
    >
    > ​ 比较运算：<=、<、==、!=、>=、>
    >
    > ​ 位运算：&、|、^（异或）、\~（位取反）
    >
    > ​ 算数运算：+、-、-（负）、\*、/、%、\*\*
    >
    > ​ 移位：<<、>>

**引用类型**

**数组：通过下标进行访问**

* **length**：当前数组长度
* **push()**：添加新的默认值到数组末尾，返回引用
* **push(x)**：数组末尾添加给点的元素
* **pop()**：从数组末尾删除元素
* **for()**：遍历
* **delete**：删除数组
  * **delete x**：它分配长度为零的动态数组或长度相同的静态数组，并将所有元素设置为其初始值
  * **delete a\[x]**：删除数组索引 x处的项目，并保持所有其他元素和数组长度不变。这尤其意味着它在数组中留有间隙

> bytes、string是一种特殊的数组
>
> bytes动态数组，类似byte\[]，但使用的gas更低
>
> string用于UTF-8数据存储

**string**

* 获取string长度：`bytes(string 参数).length`
* 字符串拼接：`string(abi.encodePacked(a,b));`
*   字符串反转

    ```solidity
    function reverse(string memory _str) external pure returns(string memory){
        bytes memory str =  bytes(_str);
        bytes memory reverse =  bytes(new string(str.length));
        for(uint i = 0; i< str.length; i ++){
        	tmp[str.length - 1 - i] = str[i];
        }
        return string(reverse);
    }
    ```

**struct（结构体）**

**定义**

```solidity
struct Todo {
    string text;
    bool completed;
}
```

**初始化**

* ```solidity
  Todo({text: _text, completed: false})
  ```
* ```solidity
  Todo memory todo;
  todo.text = _text;
  ```
* ```solidity
  Todo(_text, false)
  ```

**删除**

```solidity
delete Todo;//将重置结构体的所有成员
```

**mapping（映射）**

> 外部调用合约函数时，函数输入参数在`calldata`中，返回值在`returndata`中；
>
> 内部调用合约函数时，函数输入参数和返回值都在memory中

**定义**

```solidity
mapping(address => uint) balances
```

**变量类型**

Solidity 中有 3 种类型的变量

**局部变量**

* 在函数内部声明
* 不存储在区块链上

**状态变量**

* 在函数外声明
* 存储在区块链上

**全局变量**

**区块相关**

* **bytes32 = blockhash(uint)**：给定区块号的区块哈希，智能取得最新的256个区块的区块哈希，且不包含当前区块
* **address = block.coinbase**：挖出当前区块的矿工地址
* **uint = block.difficult**：当前区块的难度值
* **uint = block.gasglimit**：当前区块的gas限制
* **uint = block.number**：当前区块号
* **uint = block.timestamp**：当前区块的时间戳（从Uinx epoch开始的秒数）
* **tx.origin(address payable)**：最初的调用者地址

**ABI编码**

函数选择器(Function Selector)

```
abi.encodeWithSelector() = bytes4(keccak256("functionName(paramType)"))
```

一个函数调用数据的前 4 字节，指定了要调用的函数, 函数名称和参数类型 Keccak 哈希的前 4 字节，获取方式函数名称.selectorbytes4(keccak256(bytes("函数名称(参数类型)"))。

1.  参数编码

    从第5字节开始是被编码的参数，每个参数32字节(64个hex字符)，获取方式abi.encode(param);//计算参数的abi编码,会自动补齐32位 。

    bool类型的补码左侧补零

    true: 0x0000000000000000000000000000000000000000000000000000000000000001

    false：0x0000000000000000000000000000000000000000000000000000000000000000

    *   值类型和引用类型编码不同

        对于函数 sam(bytes p1, bool p2, uint\[] p3) 实际传⼊ dave、true 和 \[1, 2, 3] ，则其 ABI 编码结果将是：0xa5643bf2：函数选择器。由函数签名 sam(bytes,bool,uint256\[]) 计算得到。注意，uint 需要被替换为 uint256。0x0000000000000000000000000000000000000000000000000000000000000060：第⼀个参数（动态类型）的数据部分 的位置，即从参数编码块开始位置算起的字节数。在这⾥，是 0x60 。第二行第一个字节到第五行第一个字节的偏移量 = 960x0000000000000000000000000000000000000000000000000000000000000001：第⼆个参数：boolean 的 true。 0x00000000000000000000000000000000000000000000000000000000000000a0：第三个参数（动态类型）的数据部分 的位置，由字节数计量。在这⾥，是 0xa0。 0x0000000000000000000000000000000000000000000000000000000000000004：第⼀个参数的数据部分，以字节数组 的元素个数作为开始，在这⾥，是 4。 0x6461766500000000000000000000000000000000000000000000000000000000：第⼀个参数的内容：dave的 UTF8 编码（在这⾥等同于 ASCII 编码），并在右侧（低位）⽤ 0 值字节补充到 32 字节。 0x0000000000000000000000000000000000000000000000000000000000000003：第三个参数的数据部分，以数组的元 素个数作为开始，在这⾥，是 3。 0x0000000000000000000000000000000000000000000000000000000000000001：第三个参数的第⼀个数组元素。 0x0000000000000000000000000000000000000000000000000000000000000002：第三个参数的第⼆个数组元素。 0x0000000000000000000000000000000000000000000000000000000000000003：第三个参数的第三个数组元素。

**消息相关**

1. `uint = gasleft`：当前的剩余可用gas = 当前执行交易发送的gas - 执行所消耗的gas。
2. `bytes = msg.data`：完整的calldata字节数组。
3. `address = msg.sender`：当前消息的发送者地址。
4. `bytes4 = msg.sig`：当前的calldata的前4字节（函数选择器）。
5. `uint = msg.value`：当前消息所附带的以太币数量（以wei为单位）。
6. `uint = tx.gasprice`：触发当前消息的原始交易所制定的gas price。
7. `address = tx.origin`：触发当前消息的原始交易的发送者地址。

**异常处理**

* `assert(bool condition)`：用于检查不应该为假的代码。断言失败可能意味着存在错误。
* `require(bool condition,string … message)`：用于在执行前验证输入和条件。
*   `revert(string message)`：类似于`require`。有关详细信息，请参阅下面的代码。使用自定义错误来节省气体。

    ```solidity
    error InsufficientBalance(uint balance, uint withdrawAmount);
        function testCustomError(uint _withdrawAmount) public view {
            uint bal = address(this).balance;
            if (bal < _withdrawAmount) {
                revert InsufficientBalance({balance: bal, withdrawAmount: _withdrawAmount});
            }
        }
    ```

**算数函数**

* `addmod(uint x,uint y,uint k)`：计算任意精度的（x + y）% k。
* `mulmod(uint x,uint y,uint k)`：计算任意精度的（x \* y）% k。

**密码学函数**

* `bytes32 = keccak256(bytes memory)`：计算输入数据的Ethereum-SHA-3（Keccak256）哈希值。
* `bytes32 = sha3(bytes memory)`：等价于`keccak256(bytes memory)`。
* `bytes32 = sha256(bytes memory)`：计算输入数据的SHA-256哈希值。
* `bytes32 = ripemd160(bytes memory)`：计算输入数据的RIPEMD-160哈希值。
* `address = ecrecover(bytes32 hash,uint8 v,bytes32 r,bytes32 s)`：椭圆曲线公钥恢复。

**地址相关**

转账

* `<address>.balance`：给定地址的账户余额（以Wei为单位）。
* `<address>.transfer(uint256)`：向给定地址转账若干Wei的以太币，仅附加2300 gas，失败时直接抛出异常。
* `<address>.send(uint256)`：像给定地址转账若干Wei的以太币，仅附加2300 gas，转账出错不会抛出异常，只返回`true`/`false`，后面代码继续执行。可以在批量转账中使用。

合约调用

* `<address>.call{gas:}(bytes memory)`：向给定地址发起消息调用，附加可用gas，转账出错不会抛出异常，只返回`true`/`false`，后面代码继续执行，容易发生重入攻击。
* `<address>.callcode(bytes memory)`：等价于`call`，但保持执行上下文；即从目标地址获取相应的合约函数代码到当前合约的上下文执行；已不推荐使用，未来会移除。
* `<address>.delegatecall(bytes memory)`：上下文为调用合约的上下文，`msg.sender`、`msg.value`不变。

**合约相关**

* `this`：当前合约类型的变量，可以转换为`address`类型。
* `super`：当前合约的继承关系中上一层合约类型的变量。
* `selfdestruct(address recipient)`：销毁当前合约，并将合约余额发送到给定地址。
* `suicide(address recipient)`：等价于`selfdestruct`，但已不推荐使用。

#### 函数

**可见性**

* **Private（私有）**：限制性最强，函数只能在所定义的智能合约内部调用。
*   **Internal（内部）**：可以在所定义智能合约内部调用该函数，也可以从继承合约中调用该函数。

    合约内部调用`private`/`internal`函数，本质上是代码跳转，所以不会产生`calldata`和`returndata`，内存是公用的，msg也会保持不变；返回值是直接在栈或者内存中传递。
* **External（外部）**：只能从智能合约外部调用。 (如果要从智能合约中调用它，则必须使用 `this`。)
  * 必定会生成`calldata`和`returndata`
* **Public（公开）**：可以从任何地方调用。 (最宽松)
  * 如果直接用函数名调用，则等价于`private`/`internal`函数。
  * 如果用`this`来调用，则等价于`external`函数，会产生child message，也就是会产生`calldata`和`returndata`
  * 所以对`public`函数，如果本合约中有内部调用的话，实际产生的合约代码中会有两套处理函数接口的代码。

**状态可变性（mutability）**

* **view**：用`view`声明的函数只能读取而不能修改状态变量。
* **pure**：用`pure`声明的函数既不能读取也不能修改状态变量。
* **payable**：用`payable`声明的函数可以接受发送给合约的以太币，如果未指定，该函数将自动拒绝所有发送给它的以太币

**`receive()`函数**

* 合约最多可以具有一个`receive`函数。这个函数不能有参数，不能返回任何参数，并且必须具有`receive`可见性和 `payable`状态可变性。

当向合约发送Ether且未指定调用任何函数（calldata 为空）时执行。这是在普通的以太坊转账上执行的函数（例如，通过`send()`或`transfer()`转账）。 该函数声明如下：

```solidity
receive() external payable {
   ...
}
```

**`fallback()`函数**

​ 接受以太坊，并且`calldata`数据不为空

​ 合约最多可以具有一个`fallback`函数。

​ 在不使用内联汇编的情况下，这个函数不能有参数，不能返回任何参数，并且必须具有external可见性。如果其他函数均不匹配给定的函数签名，或者根本没有提供任何数据并且没有receive函数，则在调用合约时执行该函数

```solidity
fallback() external [payable][other modifiers]{
     ...
}
```

**modifier**

修饰符是可以在函数调用之前和/或之后运行的代码。

修饰符可用于：

* 限制访问
* 验证输入
* 防范重入黑客

modifier可以被重写，需要被重写的修改器也需要使用 `virtual` 修饰

```solidity
pragma solidity >=0.7.0 <0.9.0;

contract Base{
    modifier foo() virtual {_;}
}

contract Inherited is Base{
    modifier foo() override {_;}
}
```

如果是多重继承，所有直接父合约必须显示指定override， 例如：

```
pragma solidity >=0.7.0 <0.9.0;

contract Base1{
    modifier foo() virtual {_;}
}

contract Base2{
    modifier foo() virtual {_;}
}

contract Inherited is Base1, Base2{
    modifier foo() override(Base1, Base2) {_;}
}
```

#### 接口

接口类似于抽象合约，但是它们不能实现任何函数。还有进一步的限制：

* 无法继承其他合约,不过可以继承其他接口。
* 所有的函数都需要是 `external`
* 无法定义构造函数。
* 无法定义状态变量

接口中的函数都会隐式的标记为 `virtual` ，意味着他们会被重写。 但是不表示重写（overriding）函数可以再次重写，仅仅当重写的函数标记为 `virtual` 才可以再次重写。

#### 库

* 没有状态变量
* 不能够继承或被继承
* 不能接收以太币
* 不可以被销毁
* 如果库函数都是`internal`，库代码会嵌入到合约
* 如果库函数有`external`或`public`，库需要单独部署，并在部署合约时进行链接，使用委托调用。

#### Event

* 一种廉价的存储方式
* 可以通过Bloom Filter查询
* 外部应用可以监听区块查找对应的签名从而得知事件是否触发

```solidity
event Deposit{
	address indexed _from,
	bytes32 indexed _id,
	uint _value
}
```

每条事件记录包含主题（**topic**）和数据（data），主题为32位字节，LOG0-LOG4（操作码）。**indexed**声明可以视为主题，一个事件中可索引（**indexed**）的参数数量最多为三个，非**indexed**。

Topics\[0]：事件名称及其参数类型\*(uint256，string等)\***签名**([keccak256](https://en.wikipedia.org/wiki/SHA-3)哈希)

所有Topic都会加入交易和区块的BloomFilter，可以通过jsonrpc/web3的filter功能进行快速过滤匹配。

所有日志数据（事件参数）都是公开的，不管参数是否被声明为Indexed。

#### 合约

**创建**

可以使用`new`关键字创建合约。从 0.8.0 开始，`new`关键字`create2`通过指定`salt`选项来支持功能。

**使用create创建合约**

```go
contractAddr = Keccak256(rlp.EncodeToBytes([]interface{}{caller.Address(), nonce})[12:]
```

* 地址是由创建者地址和每次创建合约交易时的计数器(nonce)来计算合约的地址

​ new关键字将部署该合约的新实例并返回合约地址。

​ 成本较高，`CREATE`操作码目前的Gas成本为32000

**通过create2创建**

```go
contractAddr = (0xff ++ msg.sender ++ salt ++ keccak256(init_code))[12:]
```

* Create2 uses sha3(0xff ++ msg.sender ++ salt + sha3(init\_code))
* 合约的地址是由根据给定的salt，创建合约的字节码和构造函数参数来计算

**继承**

```solidity
contract K is A,B,C{…}
```

* 构造函数执行顺序：A -> B -> C -> K。
* 同名函数的覆盖顺序：K -> C -> B -> A。

#### 编译

* linkReference：当前智能合约所依赖的其他智能合约的地址的部署地址
* object：当前的智能合约字节码
* opcodes：操作代码是人类可读的低级指令
* sourceMap：ource map 是将每个合约指令与生成它的源代码部分相匹配

**操作码**

PUSH1：将1字节数据推到调用栈里

CALLDATALOAD：弹出栈上第一个值作为输入，使用值作为偏移量（msg.data\[值：值+32]）将calldata加载到栈中。栈大小32字节。

SHR：右移，弹出栈第一个值作为输入，说明右移多少位，栈第二个值代表右移的数据。右移之后会将输入压入栈。

DUP1：将栈第一个值复制再压入栈。

EQ：从栈弹出两个值，检查是否相等。相等传入1压栈，不等传入0压栈。

JUMPI：如果……，则跳转至……。从栈中弹出两个值作为输入，第一个值是跳转位置，第二个值是否执行这个跳转的bool值。1=真，0=假。

CODECOPY：将当前环境中运行的代码复制到内存

#### 内联汇编

汇编（也称为_汇编语言_）是指可使用[汇编器](https://techterms.com/definition/assembler)转换为机器代码的低级编程语言。 汇编语言与物理机或虚拟机绑定，因为它们实现了指令集。 一条指令告诉CPU执行一些基本任务，例如将两个数字相加。

以太坊虚拟机EVM有自己的指令集，该指令集中目前包含了 144个操作码，详情参考[Geth代码](https://github.com/ethereum/go-ethereum/blob/15d09038a6b1d533bc78662f2f0b77ccf03aaab0/core/vm/opcodes.go#L223-L388)

可以使用操作码直接与EVM进行交互。 这使的可以对程序（=智能合约）要执行的操作进行更精细控制

#### 操作码简介

EVM 操作码可以分为以下几类：

* 算数和比较操作
* 位操作
* 密码学计算，目前仅包含`keccak256`
* 环境操作码，主要指与区块链相关的全局信息，例如：`blockhash`或`coinbase`
* 存储、内存和栈操作
* 交易与合约调用操作
* 停机操作
* 日志操作

[Solidity 中编写内联汇编(assembly)的那些事](https://learnblockchain.cn/article/675)

​

```
push1 0x60
```

表示将1字节`0x60`放入堆栈，`PUSH1`的十六进制也是`0x60`，去掉非强制的`0x`，可以用字节码表示为`6060`

#### 状态变量存储

* 静态大小的变量（除 映射mapping 和动态数组之外的所有类型）都从位置 `0` 开始连续放置在 存储storage 中。如果可能的话，存储大小少于 32 字节的多个变量会被打包到一个 存储插槽storage slot 中。
* `struct`和数组类型的变量总会新启用一个存储槽
* 动态数组（除`bytes`和`string`外）会单独占用一个存储槽保存其元素数量，数组元素的起使位置为`keccak256(slot)`，各元素连续存放
* `bytesh`和`string`的存储会判断其长度
  * 如果他们`length <= 31`，则将数据存储在对应存储槽的高位（左对齐），在最低位中保存`length * 2`的数值；
  * 如果他们`length > 31`，则主存储槽保存`length * 2 + 1`的数值，实际数据保存在`keccak256(slot)`的位置。
* `mapping`类型的变量，其主存储槽为空，其中的key所对应的数据存储的位置为`keccak256(key.slot)`，如果这个key所对应的数据又是动态数组或者`mapping`，那么会迭代地在前面添加偏移量之后再做`keccak256`运算获取实际地位置。

```solidity
   //通过插槽位置访问对应数据
   function _get(uint i) internal pure returns (MyStruct storage s) {
        // get struct stored at slot i
        assembly {
            s.slot := i
        }
    }
```

#### 内存布局

Solidity保留了四个32字节的插槽，字节范围(包括端点)特定用途如下：

* `0x00` - `0x3f` (64 字节): 用于哈希方法的暂存空间（临时空间）
* `0x40` - `0x5f` (32 字节): 当前分配的内存大小(也作为空闲内存指针)
* `0x60` - `0x7f` (32 字节): 零位插槽

暂存空间可以在语句之间使用 (例如在内联汇编中)。 零位插槽用作动态内存数组的初始值，并且永远不应写入（空闲内存指针最初指向`0x80`）.

Solidity 总是将新对象放在空闲内存指针上，并且内存永远不会被释放(将来可能会改变)。

#### GAS

**消耗**

* 执行交易的基础gas消耗gas0 = 21000（交易基础gas消耗）+ gb（由tx.data的字节数折算的gas消耗）+ gc
  * gb：tx.data中每有一个全0字节（按照4gas/byte计算），每有一个非0字节（按照68/bytes计算）
  *   gc：如果为合约创建，+32000 + 200 \* n，n为合约运行时字节码大小

      ​ 如果为消息调用，+700，同时要转移Ether，+9000。
* EVM内存的初始大小为4个"字"，对内存的额外使用也会消耗gas，每扩展一个"字"的内存，需要消耗3gas。

**优化**

* Gas优化地本质是减少EVM操作码地数量，所以减少代码量就是优化（算法是关键）
* 能不用循环就不用循环，能不放在循环体里的处理就不要放在循环体里
* 在循环体里操作存储变量一定要尽可能优化操作次数
* 临时变量是免费的（因为是存放在栈里），虽然临时变量用多了也会增加EVM操作码的数量（因为需要用SWAPi或者DUPi操作来做栈内元素的交换或者复制），但通常这与操作存储相比要便宜的多
* 对于那些仅供其他合约调用的函数，应该声明为`external`，不要声明为`public`（因为一个函数从合约内部调用时实际上是一个跳转，不是一个`call`，不需要对参数进行ABI解码，也就是不用生成`calldata`，而所有`public`函数都需要有两套处理参数的代码）
* 将存储槽由非零值修改为零值会返还gas
* 记得打开编译器的optimize选项。

**节省**

​ 永久性存储操作码(`SSTORE`)非常昂贵。写插槽时，每个32个字节的当前成本是为20,000 Gas

*   使用短路模式排序Solidity操作

    ```
    // f(x) 是低gas成本的操作
    // g(y) 是高gas成本的操

    // 按如下排序不同gas成本的操作
    f(x) || g(y)
    f(x) && g(y)
    ```
* 删减不必要的Solidity库
* 精确声明Solidity合约函数的可见性
  *   在Solidity合约开发中，显式声明函数的可见性不仅可以提高智能合约的安全性， 同时也有利于优化合约执行的gas成本。例如，通过显式地标记函数为外部函数（External），可以强制将函数参数的存储位置设置为`calldata`，这会节约每次函数执行时所需的以太坊gas成本。

      > External 可见性比 public 消耗gas 少。
* 使用适合的数据类型
  * 在任何可以使用`uint`类型的情况下，不要使用`string`类型
  * 存储uint256要比存储uint8的gas成本低，为什么？点击这里查看[原文](https://ethereum.stackexchange.com/questions/3067/why-does-uint8-cost-more-gas-than-uint256)
    * 合约是从上到下进行存储插槽，uint128 uint128 uint256比uint128 uint256 uint128节省空间
    * Solidity合约用连续32字节的插槽来储存。当我们在一个插槽中放置多个变量，它被称为变量打包。
  * 当可以使用`bytes`类型时，不要在solidity合约种使用`byte[]`类型
  * 如果`bytes`的长度有可以预计的上限，那么尽可能改用bytes1\~bytes32这些具有固定长度的solidity类型
  * bytes32所需的gas成本要低于string类型
  * 在使用引用数据类型时
    * 结构和数组经常会被放在一个新的储存插槽中。但是他们的内部数据是可以正常打包的。一个`uint8`数组会比`uint256`数组占用更小的空间。
    *   在初始化结构时，分开赋值比一次性赋值会更有效。分开赋值使得优化器一次性更新所有变量。

        初始化结构如下：

        ```
        Point storage p = Point()
        p.x = 0;
        p.y = 0;
        ```

        而非如下：

        ```
        Point storage p = Point(0, 0);
        ```

#### 链下签名

**1. EIP-712 域哈希（Domain Hash）**

使用EIP-712，我们为ERC-20定义了一个域分隔符：

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

**2. Permit 哈希结构**

```solidity
bytes32 hashStruct = keccak256(
    abi.encode(
        keccak256("Permit(address owner,address spender,uint256 value,uint256 nonce,uint256 deadline)"),
        owner,
        spender,
        amount,
        nonces[owner],
        deadline
    )
);
```

此hashStruct将确保签名只能用于

* Permit 函数
* 从`owner`授权
* 授权`spender`
* 授权给定的`value` （金额）
* 仅在给定的`deadline`之前有效
* 仅对给定的 `nonce`有效

`nonce`可确保某人无法重播签名，即在同一合约上多次使用该签名

**3. 最终哈希**

现在我们可以用兼容 [EIP-191](https://eips.ethereum.org/EIPS/eip-191)的712哈希构建（以0x1901开头）最终签名：

```js
bytes32 hash = keccak256(
    abi.encodePacked(uint16(0x1901), eip712DomainHash, hashStruct)
);
```

**4. 验证签名**

在此哈希上，我们可以使用[ecrecover](https://solidity.readthedocs.io/en/latest/units-and-global-variables.html#mathematical-and-cryptographic-functions) 获得该函数的签名者：

```
address signer = ecrecover(hash, v, r, s);
require(signer == owner, "ERC20Permit: invalid signature");
require(signer != address(0), "ECDSA: invalid signature");
```

无效的签名将产生一个空地址，这就是最后一次检查的目的。

**5. 增加Nonce 和 授权**

现在，最后我们只需要增加所有者的Nonce并调用授权函数即可：

```
nonces[owner]++;
_approve(owner, spender, amount);
```

你可以在[此处](https://github.com/soliditylabs/ERC20-Permit/blob/main/contracts/ERC20Permit.sol)看到完整的实现示例。

#### 错误编码

1. 0x01: 如果你调用 `assert` 的参数（表达式）结果为 false 。
2.  0x11: 在`unchecked { … }`外，如果算术运算结果向上或向下溢出。

    ```solidity
    function testMin() pure public returns (int x) {
            x = type(int).min;
            - x;
        }
    ```
3. 0x12; 如果你用零当除数做除法或模运算（例如 `5 / 0` 或 `23 % 0` ）。
4. 0x21: 如果你将一个太大的数或负数值转换为一个枚举类型。
5. 0x22: 如果你访问一个没有正确编码的存储byte数组。
6. 0x31: 如果在空数组上 `.pop()` 。
7. 0x32: 如果你访问 `bytesN` 数组（或切片）的索引太大或为负数。(例如： `x[i]` 而 `i >= x.length` 或 `i < 0`).
8. 0x41: 如果你分配了太多的内内存或创建了太大的数组。
9. 0x51: 如果你调用了零初始化内部函数类型变量。

#### 编写风格

​ 函数应根据其可见性和顺序进行分组：

​ 构造函数

​ receive 函数（如果存在）

​ fallback 函数（如果存在）

​ 外部函数(external)

​ 公共函数(public)

​ 内部(internal)

​ 私有(private)

​ 在一个分组中，把 `view` 和 `pure` 函数放在最后。

​ 函数修改器的顺序应该是:

​ Visibility

​ Mutability

​ Virtual（只有标记为`virtual`的函数才可以重写它们。 如果重写后依旧是可重写的，则仍然需要标记为`virtual`）

​ Override（任何重写的函数都必须标记为`override`）

​ Custom modifiers

#### 合约升级

**问题**

**透明代理和函数选择器冲突**

​ solidity合约的函数调用都是通过函数选择器，calldata的前4个字节，对实现合约而言，完全有可能具有与代理的升级函数具有相同的函数选择器。针对这种问题可以通过管理员只能调用升级管理函数，普通用户只能调用实现合约函数，该模式被称为**透明代理模式**。如下：

```
// Sample code, do not use in production!
contract TransparentAdminUpgradeableProxy {
    address implementation;
    address admin;

    fallback() external payable {
        require(msg.sender != admin);
        implementation.delegatecall.value(msg.value)(msg.data);
    }

    function upgrade(address newImplementation) external {
        if (msg.sender != admin) fallback();
        implementation = newImplementation;
    }
}
```

**代理存储冲突和非结构化存储**

在所有代理模式变体中，代理合约都需要至少一个状态变量来保存实现合约地址。默认采用顺序存储，可能导致实现合约的存储覆盖代理合约。

**解决方式**：将实现合约slot0声明为实现地址。但是带来了问题：即要求所有委托目标合约都添加此额外的虚拟变量。这限制了可重用性，因为普通合约不能用作实现合约。这也容易出错，因为很容易忘记在合约中添加该额外变量。

为避免此问题，[非结构化存储模式](https://blog.openzeppelin.com/upgradeability-using-unstructured-storage/)被引入。

**构造函数警告**

构造函数的代码或全局变量不是已部署合约字节码的一部分。此代码仅在部署合约实例时执行一次。因此，逻辑合约构造函数中的代码将永远不会在代理状态的上下文中执行。换句话说，代理完全不知道构造函数的存在。就好像他们没有代理一样。

不过这个问题很容易解决。逻辑合约应将构造函数中的代码移动到常规的“初始化程序”函数，并在代理链接到此逻辑合约时调用此函数。需要特别注意这个初始化函数，使其只能被调用一次，这是一般编程中构造函数的属性之一。

这就是为什么当我们使用 OpenZeppelin Upgrades 创建代理时，您可以提供初始化函数的名称并传递参数。

为了确保该`initialize`函数只能被调用一次，使用了一个简单的修饰符。OpenZeppelin Upgrades 通过可扩展的合约提供此功能：

```
// contracts/MyContract.sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

import "@openzeppelin/contracts-upgradeable/proxy/utils/Initializable.sol";

contract MyContract is Initializable {
    function initialize(
        address arg1,
        uint256 arg2,
        bytes memory arg3
    ) public payable initializer {
        // "constructor" code...
    }
}
```

**实现方式**

**透明代理模式**

​ 缺点成本较高，每次调用都需要从存储中加载admin地址。

**通用可升级代理模式(UUPS)**

作为透明代理的替代。将升级逻辑放在实现合约中，而不是代理合约中。

由于代理模式采用delegatecall调用，实现合约会始终写入代理的存储中。UUPS建议所有实现合约都应继承自基础的“可代理**proxiable**”合约：

```
// Sample code, do not use in production!
contract UUPSProxy {
    address implementation;

    fallback() external payable {
        implementation.delegatecall.value(msg.value)(msg.data);
    }
}

abstract contract UUPSProxiable {
    address implementation;
    address admin;

    function upgrade(address newImplementation) external {
        require(msg.sender == admin);
        implementation = newImplementation;
    }
}
```

优点：在实现合约上定义所有函数，避免函数选择器冲突。

缺点：如果升级到没有可以升级的函数上，那么将永久锁定在该实现上，无法更改。

**非结构化存储代理**

```
|Proxy                     |Implementation           |
|--------------------------|-------------------------|
|...                       |address _owner           |
|...                       |mapping _balances        |
|...                       |uint256 _supply          |
|...                       |...                      |
|...                       |                         |
|...                       |                         |
|...                       |                         |
|...                       |                         |
|address _implementation   |                         | <=== 随机槽
|...                       |                         |
|...                       |                         |
```

EIP 1967实现随机存储

#### 智能合约安全

**重入攻击**

合约之间可以进行外部调用。同时以太坊的转账不仅仅局限于外部，合约账户也可以通过以太坊转账，且合约在接受以太坊时可能会触发`fallback`函数相关功能。

**防御方法**：

* Checks-Effects-Interactions
* 防重入锁

[**自毁函数**](https://learnblockchain.cn/article/3331)

### 流动性挖矿

#### 年利率(APR)和年溢率(APY)

**APR**：不考虑复利的影响

​ 挖矿年化APR = 挖矿每天产生的数量 \* 挖矿币价格 / LP代币价格 \* 质押LP数量 \* 365

​ 每天挖矿数量：质押LP数量 / LP池总数量 \* LP池挖矿产出数量

​ LP价格：转换成usdt价格

**APY**：考虑复利的影响

***

### Web3js

#### Web3实例

```javascript
var web3 = new Web3(Web3.currentProvider);
var web3 = new Web3(new Web3.provider.HttpProvider("http://localhost:8548"));
```

#### 数据签名

```javascript
web3.eth.sign(address,dataToSign,[callback
```

#### 发起合约调用

```javascript
web3.eth.call(callObject[,defaultBlock][,callback])
```

#### 发起合约调用/转账transfer

```javascript
web3.eth.sendTransaction(transactionObject[,callback])
```

#### 估算Gas

```javascript
web3.eth.estimateGas(callObject[,callback])
```

交易过滤

```javascript
var options = {
	fromBlock:100000,
	toBlock:1000100,
	address: "0x",
	topic: ""
};
var filter = web3.eth.folter(options);
filter.watch(function(error,log){
    consol.log(log);
});
filter.get(function(error,resukt){
    if(!error){
        console.log(result);
    }
})
```

#### 合约实例化

```javascript
var contract = web3.eth.contract(abi);
var contractInstance = contract.at(address);
var contractInstance = contract.new();
```

#### 合约事件

**实时监听（使用websocket连接）**

```javascript
let contract = new ethers.Contract(address,ABI,provider);
let filter = contract.filters.eventName();
wssprovider.on(filter,(log) => {
	console.log(log)
})
```

**获取历史事件**

```javascript
let contract = new ethers.Contract(address,ABI,provider);
let filter = contract.filters.eventName();
filter.fromBlock = 
filter.toBlock = 
let logs = await provider.getLogs(filter);
let logs = await contract.queryFilter(filterTo,-10,"latest");
```

```javascript
var event = contractInstance.event();
event.watch(function(error,result){
	if(!error){
		console.log(result)
	}
})
```

***

### Ethers

#### 链接外部库

```javascript
constant exLib = await ethers.getContractFactory("Library");
constant lib = await exLib.deploy();
await lib.deployed();
await ethers.getContractFactory("",{
	libraries:{
		Library:lib.address
	}
})
```

#### 事件

**获取历史事件**

```
let contract = new ethers.Contract(address,ABI,provider);
let filter = contract.filters.eventName()
filter.fromBlock=1000;
filter.toBlock=2000;//latest
let logs = await provider.getLogs(filter)
let logs = await contract.queryFilter(filterTo,-10,"latest")
```

**解析事件**

```javascript
const TransferEvent = new ethers.utils.Interface(["event Transfer(address indexed from,address indexed to,uint256 value)"]);
const data = Transfer.parseLog(logs[i]);
console.log("from:"+data.args.from)
console.log("from"+data.args.to)
```

```javascript
filter = {
	address:"",
	topics:{
		ethers.utils.id("eventName(type)")
	}
}
ethers.provider.on(filter,(log)=>{
	console.log(log)
})
```

***

### Harhat

#### 构建

引入开发依赖：

```
npm i -D  hardhat
```

在当前项目创建hardhat模板项目：\`

```
npx hardhat
```

安装hardhat相关插件：

```
npm install --save-dev @nomiclabs/hardhat-waffle ethereum-waffle chai @nomiclabs/hardhat-ethers ethers
```

运行hardhat本地节点：

```
npx hardhat node
```

编译

```
npx hardhat compile
```

想将 Hardhat的脚本运行在本地测试节点，只需使用`--network localhost`：

```
npx hardhat run scripts/sample-script.js --network localhost
```

#### Openzeppelin

```
npm install @openzeppelin/contracts
```

#### TypeScript支持

引入TypeScript：

```
npm install --save-dev ts-node typescript
```

引入测试包：

```
npm install --save-dev chai @types/node @types/mocha @types/chai
```

重新命名hardhat.config.js`为`hardhat.config.ts

```
mv hardhat.config.js hardhat.config.ts
```

创建[`tsconfig.json`](https://www.typescriptlang.org/docs/handbook/tsconfig-json.html)

```
{
  "compilerOptions": {
    "target": "es2018",
    "module": "commonjs",
    "strict": true,
    "esModuleInterop": true,
    "outDir": "dist"
  },
  "include": ["./scripts", "./test"],
  "files": ["./hardhat.config.ts"]
}
```

#### 验证Etherscan合同

```
npm install --save-dev @nomiclabs/hardhat-etherscan
```

并将以下语句添加到您的`hardhat.config.js`：

```
require("@nomiclabs/hardhat-etherscan");
```

或者，如果您使用的是 TypeScript，请将其添加到您的`hardhat.config.ts`：

```
import "@nomiclabs/hardhat-etherscan";
```

```javascript
npx hardhat flatten contracts/.sol >> .sol
```

### Graph

#### 安装工具

```shell
npm install -g @graphprotocol/graph-cli
```

#### 初始化子图

```shell
graph init --studio <SUBGRAPH_SLUG>
```

#### 验证

```shell
graph auth --studio <DEPLOY_KEY>
```

#### 部署

```shell
graph deploy
```
