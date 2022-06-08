# UniSwap v2

## swap

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

## addLiquidity

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
