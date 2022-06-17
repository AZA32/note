# DEX

Dex主要分为两类：订单簿、兑换池

### 兑换池

#### uniswap v2

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

**TWAP**

时间加权价格

price0CumulativeLast、price1CumulativeLast 记录 token 的累加价格

$$
priceCumulative += price * (t2 - t1)
$$

$$
TWAP = (priceCumulative2 - priceCumulative1) / (timestamp2 - timestamp1)
$$

### 订单簿

以0x协议为代表，订单链下撮合、链上结算
