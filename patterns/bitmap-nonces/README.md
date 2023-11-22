# 位图Nonces

- [📜 示例代码](./TransferRelay.sol)
- [🐞 测试](../../test/TransferRelay.t.sol)

执行一个止损订单，或一个治理提议，又或是一个元交易。。。等等这些行为有什么共同之处？它们的共同点就是这类操作都只可执行仅仅一次，不能多次。在许多主流协议中你都能找到这类操作的存在。这样的“仅执行一次”的特性是需要在链上被保证的，以防止重放攻击。想要做到这一点，许多协议会使用一种具有唯一性的标记值（nonce）并且令这个值指向一个特定的关于此交易状态存储空间，这个空间存储的数值可以指明这笔交易是否已被执行。

## 用朴素的方法尝试

如下这个合约可做为例子，它收集链下被签署过的消息，等待一段指定时间之后，在链上去执行将签署者所持有的（标准合规的）ERC20代币进行转移：

```solidity
contract TransferRelay {
    struct Message {
        address from;
        address to;
        uint256 validAfter;
        IERC20 token;
        uint256 amount;
        uint256 nonce;
    }

    mapping (address => mapping (uint256 => bool)) public isSignerNonceConsumed;

    function executeTransferMessage(
        Message calldata mess,
        uint8 v,
        bytes32 r,
        bytes32 s
    )
        external
    {
        require(mess.from != address(0), 'bad from');
        require(mess.validAfter < block.timestamp, 'not ready');
        require(!isSignerNonceConsumed[mess.from][mess.nonce], 'already consumed');
        {
            bytes32 messHash = keccak256(abi.encode(block.chainid, address(this), mess));
            require(ecrecover(messHash, v, r, s) == mess.from, 'bad signature');
        }
        // 标记这个被授权的交易状态为已被执行过
        isSignerNonceConsumed[mess.from][mess.nonce] = true;
        // 执行转移交易
        mess.token.transferFrom(address(mess.from), mess.to, mess.amount);
    }
}
```

我们期待签署人在选取 `nonce` 的值的时候能够保证在他所有签署的消息中这个值都不会被重复使用。我们的合约利用这个 `nonce` 值来锁定其对应的消息并且通过 `isSignerNonceConsumed` 映射来记录其目前的状态。这方法很直观，容易理解。。。但是我们还可以做到更进一步！

## 审视Gas成本

我们来看一下标记消息的执行状态这一操作所需要花费的gas。因为每一个 `Message.nonce` 都指向一个唯一的空间储存槽，我们就需要每一次标记状态的时候都针对一个全新的**空**储存槽进行写入操作。对于空储存槽写入要花费20k(\*) gas。为帮助理解，对比起来这相当于一笔AMM兑币交易花费gas数量的15%。尤其是对于发生频率较高的defi操作而言，这些gas成本很容易积少成多。反之，向一个*非空*的储存槽写入却仅需3k(\*) gas。利用位图nonce方式可以最小化我们需要写入空储存槽的次数，将大部分操作所耗的gas成本节省85%。

*(\*) 不算上EIP-2929中的获取冷/热状态值的成本*

## 重新来过，这次利用位图Nonces方法

仔细想想，我们完全不需要一整个32字节大小的字符串，甚至也不需要一整个8比特大小的布尔值来代表一条消息的执行状态。我们仅需要1个比特大小（`0` or `1`）足矣。所以若是我们想要将写入空储存槽的次数最小化，相对于把nonces映射到一*整个*储存槽的方式，我们大可以采用将其映射到一个储存槽内的比特位的方式。EVM的每一个储存槽都是32字节大小，所以它能容纳下256个操作的状态记录，用满之后我们才需要使用下一个新的储存槽。

![nonces slot usage](./nonces-slots.drawio.svg)

实现的方式就是通过将 `nonce` 数值用二进制表示，截出上游的248个比特位所代表的数字，将其映射至对应的唯一储存槽（这一点与前面朴素的方法相类似），然后取下游的8比特位所代表的数字用来标记这个储存槽中相应位置为`1`。如果用户是采用例如1，2，3这样从小到大递增的方式来选取 `nonce` 的而非随机选取，那么只有在他完成了255次操作之后，在第256次的时候才会再用到一个新的储存槽。

让我们在合约中实现这个方法：

```solidity
contract TransferRelay {
    // ...

    mapping (address => mapping (uint248 => uint256)) public signerNonceBitmap;

    function executeTransferMessage(
        Message calldata mess,
        uint8 v,
        bytes32 r,
        bytes32 s
    )
        external
    {
        require(mess.from != address(0), 'bad from');
        require(mess.validAfter < block.timestamp, 'not ready');
        require(!_getSignerNonceState(mess.from, mess.nonce), 'already consumed');
        {
            bytes32 messHash = keccak256(abi.encode(block.chainid, address(this), mess));
            require(ecrecover(messHash, v, r, s) == mess.from, 'bad signature');
        }
        // 标记这个被授权的交易状态为已被执行过
        _setSignerNonce(mess.from, mess.nonce);
        // 执行转移交易
        mess.token.transferFrom(address(mess.from), mess.to, mess.amount);
    }

    function _getSignerNonceState(address signer, uint256 nonce) private view returns (bool) {
        uint256 bitmap = signerNonceBitmap[signer][uint248(nonce >> 8)]; //用上游248比特位数字来映射槽的位置读取存储的值
        return bitmap & (1 << (nonce & 0xFF)) != 0; //值的二进制表示与用nonce下游8比特位位移过的1来进行按位且运算，得到相应位置是被标记成了0还是1
    }

    function _setSignerNonce(address signer, uint256 nonce) private {
        signerNonceBitmap[signer][uint248(nonce >> 8)] |= 1 << (nonce & 0xFF); //用上游248比特位数字来映射槽的位置，再用下游8比特位（可代表256个位置）来位移1进行标记，再与自身进行按位或运算
    }
}
```

## 最后的想法

- 位图nonces的方法在一些主流协议中被采用，比如Uniswap's [Permit2](https://github.com/Uniswap/permit2/blob/cc56ad0f3439c502c246fc5cfcc3db92bb8b7219/src/SignatureTransfer.sol#L142) 和 0x's [Exchange Proxy](https://github.com/0xProject/protocol/blob/e66307ba319e8c3e2a456767403298b576abc85e/contracts/zero-ex/contracts/src/features/nft_orders/ERC721OrdersFeature.sol#L662).
- 即便是你要追踪和记录的状态是大于2种的情况，也仍有办法使用位图nonces来完成。你仅仅需要增加每个操作所对应使用的比特位数量然后相应调整映射的函数表达就可以了。
- 一个有效的例子的完整代码[在这里](./TransferRelay.sol)，它的各项测试可说明其用法还有节省的gas数量可以在[这里](../../test/TransferRelay.sol)查看。
