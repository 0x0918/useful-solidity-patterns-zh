# 汇编技巧 - 1
- [🐞 测试](../../test/AssemblyTricks1.t.sol)

以下是一些简短但实用的的汇编技巧集合，它们可以帮助你节省大量的 gas，并帮助绕开一些 Solidity 本身的限制。
但是一定要非常注意使用这些技术的方式和时机，因为不恰当的使用或者是不正确的实现方式可能会导致非常严重且难以发现的错误。


## 错误冒泡 (让错误继续沿着调用栈向上传递)

以下这些方式是非常常见的向另一个合约（或 EOA）发起调用的方式，他们的共同点是被调用函数的错误回滚不会进一步调用方本身报错并回滚：

1. 使用底层调用例如 `call()`, `delegatecall()`, `staticcall()`。
2. 使用 `try`/`catch` 语句。

在这些情况下，你的代码在被调用函数报错后继续执行，返回值是调用者以 `bytes` 数组形式返回的错误数据。
你可能想要使用使用[错误处理](../error-handling)这一节中学到的技巧来处理某些错误，但如果碰到其他错误，则重新抛出。
这里比较常见的做法是简单地将 `bytes` 错误数据转换为 `string`，并将其传入 `revert()`，但这实际上是错误的，
因为 `revert` 会将传入的字符串重新编码为原生错误 `Error(string)` 的形式，从而导致了双重编码：

```solidity
// This is the WRONG way to re-throw raw revert data (`revertBytes`) because it
// re-encodes the revert data as an `Error(string)` revert type. 
revert(string(revertBytes))
```

相反，这里可以考虑直接使用汇编来抛出实际的错误数据（要注意的是，这仅在 `revertBytes` 存在内存中时可行，
当然绝大部分情况它都是存在内存中的）：

```solidity
// Re-throw `revertBytes` revert data as-is.
assembly { revert(add(revertBytes, 0x20), mload(revertBytes)) }
```

类似的，在 `try/catch` 中使用也很简单！

```solidity
try otherContract.foo() {
    // handle successful call...
} catch (bytes memory revertBytes) {
    // call failed, do some error processing
    // if all else fails, bubble up the revert
    assembly { revert(add(revertBytes, 0x20), mload(revertBytes)) }
}
```

## 计算两个单独的 32 字节类型的哈希值

现实中经常会遇到需要对两个 32 字节值进行哈希的情况，比如在 [traversing merkle trees](../merkle-proofs/) 中。
由于 `keccak256()`（内置哈希函数）接收字节数组，所以一般会想到先使用 `abi.encode()` 将两个单词进行拼接后再做哈希：

```solidity
uint256 word1 = ...;
bytes32 word2 = ...;
// Concatenate `word1` and `word2` then compute their hash.
bytes32 hash = keccak256(abi.encode(word1, word2));
```

上述操作对于任意数据类型和数量都是有效的，但如果你只需要哈希两个 32 字节的值（或者组合起来是 64 字节的值），你可以使用一些简单的汇编以更小的 gas 成本来做到同样的事情：


```solidity
bytes32 hash;
assembly {
    mstore(0x00, word1)
    mstore(0x20, word2)
    hash := keccak256(0x00, 0x40)
}
```

这比常规的 `keccak256(abi.encode())` 成本更小是因为 `abi.encode()` 会分配一个新的内存缓冲区，
然后用这个缓冲区来把两个值拼接到一起，最后再传给 `keccak256()`。而汇编版本只是把这两个值拼接在内存的头两个槽位中（0x00-0x40），这在 EVM 的规范中是免费可用的临时空间，从而避免了分配新的内存导致的内存扩展带来的 gas 开销。


## 在两种数组类型之间进行转换

Solidity 不允许你直接转换不同元素类型的数组（`bytes` 和 `string` 类型是一个例外）。如果你在你的项目中导入第三方库，你有时会遇到这样的情况，一个导入的函数接收的数组类型和你在你的代码中使用的数组类型不同，但你知道它们在底层其实是兼容的。这些例子包括：

- `address[]` vs `address payable[]`
- `address[]` vs `interface[]`
- `address[]` vs `contract[]`
- `uint160[]` vs `address[]`
- `uint256[]` vs `bytes32[]`
- `uint256[N]` vs `bytes32[N]` 
- etc.

最直观的，你可以通过遍历整个数组来对每个元素进行转换，但是因为转换过程还额外复制了每个元素，所以 gas 开销很大：

```solidity
// Doing a conversion between compatible array types the hard way.
address[] memory addressArr = ...;
IERC20[] memory erc20Arr = new IERC20[](addressArr.length);
for (uint256 i; i < addressArr.length; ++i) {
    erc20Arr[i] = addressArr[i];
}
```

实际上，变量本身只是存储在堆栈上的指针, 它所指向的内存才是 `memory` 数组数据，使用汇编可以让你直接更新某个变量中存储的指针。在这个例子中只需要新建一个目标类型的数组变量然后将其指针指向原数组在 `memory` 中的位置即可。因此，你可以使用以下方式很容易地满足前面例子的要求：


```solidity
// Cheaply cast between compatible dynamic arrays. 
address[] memory addressArr = ...;
IERC20Token[] memory erc20Arr; // No need to allocate new memory.
// Point `erc20Arr` to the same location as `addressArr`
assembly { erc20Arr := addressArr }
```

该方式在静态内存数组之间也可以工作，但相较动态数组，这种场景下带来的 gas 收益会小一些，因为声明静态数组时 EVM 会立即为它分配新的内存：

```solidity
// Cheaply cast between compatible statically sized arrays. 
address[3] memory addressArr = ...;
IERC20Token[3] memory erc20Arr;
// Point `erc20Arr` to the same location as `addressArr`
assembly { erc20Arr := addressArr }
```

注意，这些方法不适用于 `calldata` 数组，因为他们的指针对应的底层实现是完全不同的。

## 不同类型的结构体之间的转换

你也可以使用前面的数组转换技巧在*兼容*  `memory` 结构体之间转换：

```solidity
struct Foo {
    address addr;
    uint256 x;
}

// All fields in `Bar` are bit-compatible with `Foo`. 
struct Bar {
    IERC20 erc20;
    bytes32 x;
}

Foo memory foo = MyStruct({...});
// Point `bar` to the contents of `foo`.
Bar memory bar;
assembly { bar := foo }
```

结构体和静态数组在内存分配方式上实际上是类似的，所以这种方法会产生与静态数组一样的内存扩展带来的额外 gas 开销，但相比一个字段一个字段地手动转换，还是节省了不少复制操作带来的开销。


## 缩短动态内存数组的长度
动态大小的内存数组变量指向的内存位置的前 32 字节包含了数组的长度，数组中的元素紧随其后。

```
                               ┌────────────────────────┐
arr = new uint256[N]() ───────►│      Length = N        │  ptr + 0x00
                               ├────────────────────────┤
                               │      Element 0         │  ptr + 0x20
                               ├────────────────────────┤
                               │      Element 1         │  ptr + 0x40
                               ├────────────────────────┤
                               │         ...            │
                               ├────────────────────────┤
                               │      Element N         │  ptr + 0x20 * N
                               └────────────────────────┘
```


使用汇编，你可以直接写入这个位置来改变存储的数组长度！ ⚠️ 请记住，通常只有缩小数组大小才是安全的，因为扩大数组可能会导致你读写已经为其他变量预留的内存位置 ⚠️。


```solidity
uint256[] memory arr = new uint256[](100);
assert(arr.length == 100);
// Shorten the `arr` dynamic array by 1 (ignoring the last element).
assembly { mstore(arr, 99) }
assert(arr.length == 99);
```

这种方式修改了原数组的长度，所以需要确保相关的代码逻辑是否需要依赖数组长度保持不变。


## 缩短静态长度的内存数组

静态大小的数组*不会*在指针指向的位置开头数组存储长度，因为长度在编译时就已经确定了，所以上述方法对它们不起作用。但是你可以使用数组转换技巧来创建一个固定长度的引用，这个引用是原始数组的一个子集。虽然同样需要声明一个新的静态数组变量，因此无可避免的会扩展内存，但仍然可以节省复制每个元素带来的 gas 开销：

```solidity
uint256[10] memory arr;
// Shorten the `arr` fixed array by 1 (ignoring the last element).
uint256[9] memory shortArr;
assembly { shortArr := arr }
```


因为静态大小的数组没有长度前缀，所以你甚至可以将新变量指向原始数组内的一个偏移位置，以此来创建一个共享切片！

```solidity
uint256[10] memory arr;
// Create a shared slice of the original array, starting at the 2nd (idx 1) element to the 9th (idx 8).
uint256[8] memory shortArr;
assembly { shortArr := add(arr, 0x20) }
```

