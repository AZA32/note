# DEX

Dex主要分为两类：订单簿、兑换池

### 兑换池

#### Uniswap

**V2**

AMM：AutoMated Market Maker

* AutoMated：自动，没有中间机构进行资金交易
* Market Maker：做市商（保证订单得以执行），流动着提供者（LP）
  * 流动性指是如何快速和无缝的购买或出售一项资产
  * LP是提供资产的人以实现快速交易

常量乘机模型：k=x\*y

* x：token0的储备量
* y：token1的储备量

**提供流动性**

**swap**

> k值保持不变
>
> * 减少reserve0，就必须增加reserve1
> * 减少reserve1，就必须增加reserve0

设储备量A = x,储备量B=y

swap\~\~\~ForExactToken调用`getAmountIn`根据输入计算输出公式：

$$
(x + 0.997dx)(y - dy) = xy; dy(x + 0.97dx) = 0.997dxy; dy = y * dx * 997 / (x * 1000) + dx * 997; 输出 = y * 输入 * 997 / (x * 1000 + 输入 * 997)
$$

swapExactToken\~\~\~调用`getAmountOut`根据输出计算输入公式：

$$
(y - dy)(x + dx0.997) = xy; 0.997dx(y - dy) = dyx; dx = 1000 * dy * x / (y - dy) * 997
$$

***

**addLiquidity**

> 转入token0、token1，增加reserve0、reserve1，获取流动性凭证

添加流动性数量计算：

$$
输入B数量 = 输入A * 储备量B / 储备量A
$$

上述条件不满足时，即输入B精确数量小于大于输入B期望数量，再进行下列公式判断

$$
输入A数量 = 输入B * 储备量A / 储备量B
$$

流动性token计算数量：

$$
liquidity = Math.min(amount0.mul(_totalSupply) / _reserve0, amount1.mul(_totalSupply) / _reserve1);
$$

**remove**

> 添加流动性凭证，撤出token0、token1

**TWAP（时间加权价格）**

price0CumulativeLast、price1CumulativeLast 记录 token 的累加价格

$$
priceCumulative += price * (t2 - t1)
$$

$$
TWAP = (priceCumulative2 - priceCumulative1) / (timestamp2 - timestamp1)
$$

#### Sushiswap

**流动性挖矿**

在流动性提供者之间公平地分配代币，并且随着时间的推移线性分配——比如说，以 R 每秒

池切割成单秒时间片。对于这些切片中的每一个，给定的流动性提供者应该收到

$$
R\cdot \frac{l(t)}{L(t)}
$$

> R：单位时间内分配池子的代币总量
>
> l(t)：t时刻流动性提供者提供的流动性代币余额
>
> L(t)：t时刻该池的质押流动性总量t

奖励从t0至t1将是他们每一秒的奖励总和：

$$
\sum_{t=t_0}^{t_1} R\cdot \frac{l(t)}{L(t)}
$$

如果用户的余额在这段时间内保持不变，则上述公式可以简化为：

$$
R\cdot l \cdot \sum_{t=t_0}^{t_1}\frac{1}{L(t)}
$$

然后再将从t0到t1的时间段可以拆分成\[0,t0],\[0,t1]两个时间段之差，公式可变形为：

$$
R\cdot l \cdot(\sum_{t=0}^{t_1} \frac{1}{L(t)}-\sum_{t=0}^{t_0}\frac{1}{L(t)})
$$

**accSushiPerShare（单位份额奖励）**

MasterChefV2合约中质押、提取、收割都会调用updatePool函数进行更新当前池子的accSushiPerShare

```solidity
if (lpSupply > 0) {
    uint256 blocks = block.number.sub(pool.lastRewardBlock);
    uint256 sushiReward = blocks.mul(sushiPerBlock()).mul(pool.allocPoint) / totalAllocPoint;
    pool.accSushiPerShare = pool.accSushiPerShare.add((sushiReward.mul(ACC_SUSHI_PRECISION) / lpSupply).to128());
}
```

$$
R = sushiReward
$$

$$
L(t) = lpSupply
$$

$$
accSushiPerShare = R \cdot \sum_{t=0}^{t_1}\frac{1}{L(t)}
$$

**rewardDebt（用户已奖励）**

质押时计算公式，用户提供流动性的那一个时刻为t(0)

```solidity
user.rewardDebt = user.rewardDebt.add(int256(amount.mul(pool.accSushiPerShare) / ACC_SUSHI_PRECISION));
```

$$
l = amount
$$

$$
rewardDebt = l \cdot accSushiPerShare = l \cdot R \cdot \sum_{t=0}^{t_0}\frac{1}{L(t)}
$$

**\_pendingSushi（奖励）**

收取时时间为t1

```solidity
int256 accumulatedSushi = int256(user.amount.mul(pool.accSushiPerShare) / ACC_SUSHI_PRECISION);
uint256 _pendingSushi = accumulatedSushi.sub(user.rewardDebt).toUInt256();
```

$$
Reward=l\cdot accSushiPerShare - rewardDebt=l\cdot(R\cdot\sum_{t=0}^{t_1}\frac{1}{L(t)}-R\cdot\sum_{t=0}^{t_0}\frac{1}{L(t)})
$$

### 订单簿

以0x协议为代表，订单链下撮合、链上结算
