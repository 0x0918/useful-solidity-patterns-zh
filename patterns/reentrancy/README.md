# 可重入性

- [📜 示例代码](./AppleDAO.sol)
- [🐞 测试](../../test/AppleDAO.t.sol)

几乎所有的合约都会或多或少地以某种直接或间接形式向外部调用函数，所被调用的合约有可能是不可信，不可控的。每当外部调用发生时，代码执行的控制权就被移交给外部合约。在以太坊合约里不存在并行处理的概念，所以每当自身的执行尚未完成时控制权被交出，那么合约都会等待外部函数返回后再继续自身代码的执行，这样就向著名的重入攻击大开方便之门。

External calls can come in the obvious form of a simple call to a function on a contract, the less obvious transfer of ETH to an address (which is just an empty call), deploying a contract, or the transfer of tokens with a callback or hook mechanism (such as ERC777 and ERC721).

## 苹果全部都被卷走

假设你的合约就是Alice，职能就是给人们发苹果，每人只给发一个。但是Alice的记忆像金鱼一样差，必须要将所有事情写下来才能够记住。那么，现在贪心的Bob过来了，他其实想要*2个苹果*! 他该怎么愚弄Alice来得到2个苹果呢？

1. Bob管Alice要1个苹果。
2. Alice查她的账本看之前是否已经给过Bob1个苹果，账本上没有写，于是她给Bob1个苹果。
    1. 在Alice把这1个苹果记账*之前*，Bob立即又管Alice要1个苹果。
    2. Alice低头看账本，账本上没有写她以前给过Bob苹果，于是她又给Bob1个苹果。
    3. 现在Alice在账本上记下已经给了Bob1个苹果。
3. 现在Alice（又）在账本上记下已经给了Bob1个苹果。

进一步想，如果Bob想要拿走Alice所有的苹果，他仅仅需要将自己的请求无限嵌套，在Alice有机会进行记账之前就先把她的苹果都卷跑了。这正是那次臭名昭著的[DAO hack](https://www.immunebytes.com/blog/an-insight-into-the-dao-attack/)的操作手法。

将Alice和Bob的角色放在Solidity写成的智能合约里他们看起来会是什么样子的？假设那些苹果就是ERC721 NFTs，由Alice负责铸币。

```solidity
contract Apples is ERC721("Apples") { /* ... */ }

contract Alice {
    Apples public immutable APPLES = new Apples();
    mapping (address => boolean) _hasReceivedApple;

    function claimApple() external {
        require(!_hasReceivedApple[msg.sender]);
        // safeMint() 调用接收方的onERC721Received() 处理函数。
        APPLES.safeMint(msg.sender);
        _hasReceivedApple[msg.sender] = true;
    }
}

contract Bob {
    function exploit(Alice alice) external {
        _claim(alice);
    }

    function onERC721Received(address operator, address, uint256, bytes calldata) external {
        _claim(Alice(operator));
        return this.onERC721Received.selector;
    }

    function _claim(Alice alice) private {
        // 等我们有了100个苹果之后就不继续管Alice要了。
        if (alice.APPLES().balanceOf(address(this)) < 100) {
            alice.claimApple();
        }    
    }
}

```

*⚠️ 请注意上面仅仅是一种非常简化的重入攻击的实现的例子。实际上重入攻击可以有多种很不同的实现方式，有时候也会用到一些中间起桥接作用的合约/角色。还有一点很重要那就是，重入攻击通常是针对某一单个合约而言，但也有的情况下它是在整个协议层面上起作用如果它的操作是跨多个合约（甚至跨多个协议）的话。重入攻击也可以（常常也是）去连带攻击多个不同的函数，如果这些函数都共同依赖某些状态变量的话。*

## 保护你的苹果

让我们来看一下我们应该怎样通过两种不同但都是常见的防御机制/模式来保证Alice不被恶意利用。

### “先检查-再更新状态-最后交互”的模式
“先检查-再更新状态-最后交互”的模式（简称“CEI”模式）算是个顺口溜，来表达一种对合约代码逻辑的设计来减少被重入攻击的风险。它还能带来另一个额外的好处，就是能让你的代码更易懂，所以无论你认为重入攻击对你的合约有或没有影响你都应该考虑采用这个模式来写代码。这个模式可以说是都嵌在大多数资深solidity码农的骨子里了，写这种风格的代码就像是条件反射一样自然。

此模式采用逻辑顺序如下:

1. **检查**: 检验函数或操作的输入变量，准入权限还有初始状态。
2. **更新状态**: 在心里先过一遍交互逻辑，然后把预计会产生的状态变化先作为结果在账本中记录下来。
3. **交互**: 在这一步再去真正调用外部函数做交互，转移资产等等。

因为调用外部函数是你的执行逻辑里的最后一步，你的代码执行就不会卡在一个未完成的状态中而先将执行权移交出去了。

再来看Alice的例子，她的执行逻辑里这三步都有，但是顺序错了。她做的是“检查-交互-更新状态”，而非“检查-更新状态-交互”。若是把顺序纠正过来，她就不会暴露于重入攻击风险之下了，因为她会在Bob有机会重复提出要求之前先将“Bob已收到1个苹果”的结果记入账中。

```solidity
contract Alice {
    ...
    function claimApple() external {
        // 检查: Bob还没收到过苹果。
        require(!_hasReceivedApple[msg.sender]);
        // 更改状态: Bob已经要过了1个苹果。
        _hasReceivedApple[msg.sender] = true;
        // 交互: 给Bob1个苹果。
        APPLES.safeMint(msg.sender);
    }
}

```

### 重入守卫 (互斥锁)
有时候你没法将你的逻辑真的去写成“CEI”的顺序。因为有可能你需要依赖一个外部的交互的输出值来计算一些变量的最终状态会是什么，这种情况你无法提前预知或独自计算出最终状态，那么你可以改成使用重入守卫来防止重入攻击。

重入守卫实质上就是一个临时的状态变量，它代表了某一个操作正在进行中，像开关的指示灯一样，当灯亮的时段内，与这个进行中的操作所互斥的其他操作（或者重启自身操作）是不允许被执行的，直到那一个操作完成，灯灭，方可进行其他。很多合约采用一个专用的链上状态变量充当这个互斥锁角色(查看这里 [standard OpenZeppelin implementation](https://docs.openzeppelin.com/contracts/4.x/api/security#ReentrancyGuard))，并将它适用于任何可能暴露于风险中的函数内。重入守卫常常被写成一个修饰函数，它先检验标记状态（灯是否是灭的），然后修改标记状态（开灯），执行预设的代码逻辑，最后重置标记状态（关灯）。

如果Alice有了重入守卫就会变成如下这样：


```solidity
contract Alice {
    ...
    bool private _reentrancyGuard;

    modifier nonReentrant() {
        require(!_reentrancyGuard);
        _reentrancyGuard = true;
        _;
        _reentrancyGuard = false;
    }

    // 还是旧例子中的有漏洞的代码没有改，但是这次加上了重入守卫。
    function claimApple() external nonReentrant {
        require(!_hasReceivedApple[msg.sender]);
        APPLES.safeMint(msg.sender);
        _hasReceivedApple[msg.sender] = true;

    }
}
```

现在如果Bob想要利用嵌套的手段来在Alice的执行尚未完成之前重复调用 `claimApple()`，那个修饰函数就会看到重入守卫是活跃状态（灯是亮的），这个调用就会被逆转。

采用重入守卫的方法是非常方便的，不用太费脑筋就可实现，所以它也是个很流行的方案。然而，它也带来一些顾虑。

- 重入守卫是一个自己占有储存空间的链上状态变量。向链上的储存空间进行写入操作是要花费gas的，尤其是向一个新的空间写入更贵。尽管大部分费用可被退回，因为修饰函数会将那个空间内的值重置回原值，它也是会导致gas上限的估算增加，用户可能会感到意外。
    - 有时候你可以避免使用一个专用的链上状态变量来充当重入守卫。你可以去循环利用某个状态变量，这个状态变量应是那种在这个操作中注定要被修改状态的那种，那么你就可以先检查它，然后把它修改成一个预设的代表着无效状态的值（等同于灯亮），然后执行逻辑，在最后再把它的值改成逻辑完成之后它应当成为的结果值。这过程中它所起的防卫效果跟专用的重入守卫是类似的。
- 重入守卫的这种朴素的实现方法是只能在单一合约中起作用的。如果一个复合型的协议是跨越多个合约的，并且在这些合约中有些操作是需要互斥的，那么在这个情况下你就需要想一个办法来让重入守卫的状态能被系统中的其他合约全都能看到。

## 例子
这个 [例子](./AppleDAO.sol) 是上述所讲的全部内容和解决方案的一个完整的实现方式。为了简约的表达，采用了ERC721代币标准的删节简化版本。你可以查看这些 [测试](.../../test/AppleDAO.t.sol) 使用 `forge test -vvvv --match-path test/AppleDAO.t.sol` 命令行来加深理解执行流程。
