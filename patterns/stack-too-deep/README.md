# 堆栈过深的解决方案
- [📜 示例代码](./StackTooDeep.sol)
- [🐞 测试](../../test/StackTooDeep.t.sol)

EVM 是基于栈的，这意味着大多数 EVM 指令将从栈顶取出参数，而栈是一种小型的、独特内存，用于存储工作数据（通常是局部变量）。

```
    ┌───────────────────┐
n+2 │        b          ├─────────┐                    ┌───────────────────┐
    ├───────────────────┤         ├──► ADD(a,b)───────►│       a + b       │ n+1
n+1 │        a          ├─────────┘                    ├───────────────────┤
    ├───────────────────┤                              │        ...        │  n
 n  │        ...        │                              └───────────────────┘
    └───────────────────┘                                    le stack
           le stack
```

在 Solidity 中，默认情况下，大多数函数参数和变量也会存放在栈中，并在执行过程中根据需要进行洗牌。栈总共可容纳 1024 个 32 字节的值，但在任何时候都只能直接访问顶部的 32 个槽位对应的值。因此，在复杂的函数中，很容易出现编译失败的情况，因为编译器会遇到栈不可访问的变量。

![compiler error](./error.png)

让我们来看看解决这一问题的常见方法。

## 使用 IR 代码生成器进行编译

新版本的 `solc`（Solidity 编译器）建议使用 `--via-ir` 参数，在优化之前先将 Solidity 代码编译成 [yul](https://docs.soliditylang.org/en/v0.8.17/yul.html)。IR 代码生成路径能够自主地将栈变量移入"内存"（内存容量大且可自由寻址），以绕过栈大小限制。[Foundry](https://book.getfoundry.sh/config/) 和 [Hardhat](https://hardhat.org/hardhat-runner/docs/guides/compile-contracts) 都能传递此参数。

这种解决方案只需你做很少的一部分工作，但可能无法解决所有情况，而且可能会增加编译时间，并带来与不太成熟的代码生成流水线相关的风险。无论如何，采用上述方法中的一种也可能会产生更清晰、更易于人工验证的代码，所以不要止步于此！

## 块作用域

在复杂的函数中，并不是每个声明的变量在整个函数体中都是被需要的。通过保持变量声明与实际使用它们的代码之间的距离这一良好的编码习惯，通常可以识别出在函数中生命周期有限的变量。然后，你可以使用花括号将这些声明和操作包裹起来（`{ ... }`），这样编译器就可以提前丢弃这些变量。任何需要在逻辑之外持久存在的变量都应在块作用域之外声明。

请看下面的例子：

```solidity
uint16 feeRateBps = manager.getFeeRate();
uint256 exchangeRate = getExchangeRate(fromToken, toToken);
if (exchangeRate == 0) revert ZeroExchangeRateError();

uint256 toAmount = (fromAmount * exchangeRate) / 1e18;
uint256 feeAmount = (toAmount * feeRateBps) / 1e4;
toAmount -= feeAmount;

fees[toToken][toAmount] += feeAmount;
balances[fromToken][user] -= fromAmount;
balances[toToken][user] += toAmount;

return toAmount;
```

在代码块结束时，有 4 个局部变量被推入栈中。但是，只要稍加重新排列和使用块作用域，我们就可以只推入 1 个变量（`toAmount`）：

```solidity
uint256 toAmount;
{
   uint256 exchangeRate = getExchangeRate(fromToken, toToken);
   if (exchangeRate == 0) revert ZeroExchangeRateError();
   toAmount = (fromAmount * exchangeRate) / 1e18;
   {
      uint16 feeRateBps = manager.getFeeRate();
      uint256 feeAmount = (toAmount * feeRateBps) / 1e4;
      toAmount -= feeAmount;
      fees[toToken][toAmount] += feeAmount;
   }
}

balances[fromToken][user] -= fromAmount;
balances[toToken][user] += toAmount;

return toAmount;
```

该版本的代码也很容易验证，因为你确切地知道局部变量用于哪些操作，以及何时可以安全地释放它。仅出于这个原因，即使没有遇到堆栈问题，也应该更多地使用块作用域！

## 使用内存结构体

你可以手动声明存放在 `memory` 中的变量，而不是通过定义 `struct` 将变量存储到堆栈中。使用这种方式时，堆栈中唯一需要存储的便是指向结构体的 `memory` 偏移量的指针，因此多个变量可以有效地合并为一个堆栈项。

```solidity
struct ExchangeVars {
   uint16 feeRateBps;
   uint256 exchangeRate;
   uint256 toAmount;
   uint256 feeAmount;
}

...

ExchangeVars memory vars;
vars.feeRateBps = manager.getFeeRate
vars.exchangeRate = getExchangeRate(fromToken, toToken);
... // etc
```

但这种方法真正最有意义的地方，是在接收或返回多个参数的函数中，因为它提供了即时的可读性和可维护性。巧合的是，接收或返回多个参数的函数经常会遇到堆栈过深的问题，因为每个参数都会占用一个堆栈空间。

为了说明这一点，让我们调整下面的函数：

```solidity
function _computeExchangeAmounts(
   uint256 toAmount,
   uint256 exchangeRate,
   uint16 feeRateBps
)
   private
   pure
   returns (uint256 feeAmount, uint256 toAmount) 
{
   toAmount = (fromAmount * exchangeRate) / 1e18;
   feeAmount = (toAmount * feeRateBps) / 1e4;
   toAmount -= feeAmount;
}
```

改为通过 struct 传递参数，从而减少堆栈空间的使用：

```solidity
struct ComputeExchangeAmountsArgs {
   uint256 toAmount;
   uint256 exchangeRate;
   uint16 feeRateBps;
}

function _computeExchangeAmounts(ComputeExchangeAmountsArgs memory args)
   private
   pure
   returns (uint256 feeAmount, uint256 toAmount) 
{
   toAmount = (args.fromAmount * args.exchangeRate) / 1e18;
   feeAmount = (toAmount * args.feeRateBps) / 1e4;
   toAmount -= feeAmount;
}
```

调用包含大量参数的函数时容易出错，尤其是在多个参数类型兼容的情况下。想想看，如果在重构过程中调换了参数顺序，而没有考虑到所有调用的地方，会发生什么情况😱！有了结构体参数，你就可以利用有名字段来避免依赖参数的顺序，而且每个参数的含义也会更加清晰。因此，即使没有堆栈问题，这也是一种值得采用的模式。

```solidity
// Original function call. Argument order matters 🤔:
... = _computeExchangeAmounts(toAmount, getExchangeRate(fromToken, toToken), manager.getFeeRate());

// New function call with named struct field initializers. Independent of argument order 😏:
... = _computeExchangeAmounts(ComputeExchangeAmountsArgs({
   toAmount: toAmount,
   exchangeRate: getExchangeRate(fromToken, toToken),
   feeRateBps: manager.getFeeRate()
}));
```

## 更多奇技淫巧

在极端情况下，你可能需要采用一些奇技淫巧。你大概不会遇到这些情况，所以我只简单介绍一下。

### 进入新的执行上下文

任何时候进行外部函数调用（调用另一个合约或使用 `this.fn()` 语法），都会进入一个新的执行上下文，它带有一个全新的（几乎为空）堆栈。当然，外部调用会产生一些开销，但加入 [EIP-2929](https://eips.ethereum.org/EIPS/eip-2929) 后，开销大大降低了（例如，如果你调用自身）。虽不雅观，但偶尔也可行。

### （临时）使用 `storage`
`storage` 的读取和写入是出了名的开销大，所以这不是一个好的答案。但是，`storage` 不会受到 `memory` 扩展的[二次 gas 成本](https://github.com/wolflo/evm-opcodes/blob/main/gas.md#a0-1-memory-expansion)的影响。因此，当与 [EIP-2200](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-2200.md) 退款结合使用时，该情况也许会适用🙃。

### 链下计算
有时，合约的大部分计算都可以在链下进行，计算结果只需通过顶层调用传入即可，合约只需执行更简单的验证步骤即可。例如，与其在链上对排序数据进行二分搜索，不如在链下进行搜索，然后将索引传入，合约只需验证其与相邻数据的有效性即可。

### 通过位运算打包
每个堆栈值，无论它实际声明为哪种变量类型，最终都将占用一个完整的 32 字节槽。因此，如果你不关心精度（例如，`uint128` 与 `uint256`），你可以通过一些位运算将多个值打包到一个 `uint256` 或 `bytes32` 中。当然，这样做会大大降低可读性，而且在交互过程中极易出错，因此请谨慎使用这种方法。