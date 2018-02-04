> 源文章来源于[Diving Into The Ethereum VM](https://medium.com/@hayeah/diving-into-the-ethereum-vm-6e8d5d2f3c30),作者 [@Howard](https://medium.com/@hayeah?source=post_header_lockup) 拥有所有版权信息.欢迎在 Medium 关注他有关区块链以及以太坊的文章

Solidity 提供了许多高级的语言抽象， 但这些特性也使得理解我的程序在运行的时候究竟发生了什么变得困难。 阅读 Solidity 的文档仍旧无法解决我对于一些最基本事物的困惑。

string， bytes32， byte[ ]， bytes 这四种类型之间有什么不同?

- 何时使用哪一个 ?
- 当我将 string 类型转换为 bytes 类型的时候发生了什么? 我能转换成 byte[ ] 类型吗 ?
- 转换的时候有多少开销 ?

EVM 是如何存储映射关系的?
- 为什么我不能删除一个映射关系 ？
- 我能够映射到一个映射吗? ( 答案是可以， 但 EVM 是如何做到的 ? )
- 为什么只有存储型映射， 而没有内存型映射 ？

编译后的合约是如何关联到 EVM 的
- 一个合约是如何被创造的 ?
- 什么是一个构造器 ?
- 什么是回调函数 ?

出于以下几个理由， 我认为学习像 Solidity 这样的高级语言是如何在以太坊虚拟机( EVM ) 运行的是一件很好的投资
1.  Solidity 不是最后一个 EVM 语言， 一些更好的 EVM 语言正在不断得涌现当中。


2.  EVM 是一个数据库引擎。要理解智能合约在任意一个 EVM 语言中是如何运行的， 你必须得理解数据是如何被组织， 存储以及维护的。

3.  知道如何成为贡献者。以太坊的工具链还处于初级阶段， 对 EVM 的充分了解能够帮助你制造优秀的工具给自己和别人。
4.  智力上的挑战。 EVM 的设计会涉及到加密， 数据结构， 程序语言设计等技术， 这将会是一个很好的学习机会。


在这一系列文章中， 我将会解构一些简单的 Solidity 合约， 理解他们是如何以 EVM 的字节码工作的。

我希望学到和写作的内容有：
- EVM 字节码的基础。
- 不同的类型(映射， 数组)是如何被表示的。
- 当一份合约被创造时发生了什么。
- 当一个方法被调用时发生了什么。
- ABI(二进制应用接口)是如何桥接不同的 EVM 语言的。
 
我的最终目标是能够完全理解一个被编译后的 Solidity 合约。接下来让我们以阅读一些基本的 EVM 字节码开始吧 !

这份 [EVM 指令集](https：//gist。github。com/hayeah/bd37a123c02fecffbe629bf98a8391df) 的表格可能是一个有用的参考。

## 一个简单的合约
我们的第一个合约有一个构造器， 以及一个状态变量：
```
// c1.sol
pragma solidity ^0.4.11;
contract C {
    uint256 a;
    function C() {
      a = 1;
    }
}
```

使用  solc  编译这个合约：

```
$ solc --bin --asm c1.sol
======= c1.sol:C =======
EVM assembly:
    /* "c1.sol":26:94  contract C {... */
  mstore(0x40, 0x60)
    /* "c1.sol":59:92  function C() {... */
  jumpi(tag_1, iszero(callvalue))
  0x0
  dup1
  revert
tag_1:
tag_2:
    /* "c1.sol":84:85  1 */
  0x1
    /* "c1.sol":80:81  a */
  0x0
    /* "c1.sol":80:85  a = 1 */
  dup2
  swap1
  sstore
  pop
    /* "c1.sol":59:92  function C() {... */
tag_3:
    /* "c1.sol":26:94  contract C {... */
tag_4:
  dataSize(sub_0)
  dup1
  dataOffset(sub_0)
  0x0
  codecopy
  0x0
  return
stop
sub_0: assembly {
        /* "c1.sol":26:94  contract C {... */
      mstore(0x40, 0x60)
    tag_1:
      0x0
      dup1
      revert
auxdata: 0xa165627a7a72305820af3193f6fd31031a0e0d2de1ad2c27352b1ce081b4f3c92b5650ca4dd542bb770029
}
Binary:
60606040523415600e57600080fd5b5b60016000819055505b5b60368060266000396000f30060606040525b600080fd00a165627a7a72305820af3193f6fd31031a0e0d2de1ad2c27352b1ce081b4f3c92b5650ca4dd542bb770029

```


```6060604052...```这个数字就是 EVM 运行的时候实际运行的字节码 

## 逐步深入
在大多数 Solidity 程序中，有一半的汇编编译都是类似的。我们将会在下面的步骤中查看他们。现在我们检查下我们的合约中的独特部分， 即普通的存储变量赋值 ：

```a = 1```

这个赋值过程会被表示为字节码 6001600081905550。 将它拆解为一行一条指令表示：
```
60 01
60 00
81
90
55
50
```
EVM 实质上是一个从上到下执行每一条指令的循环。将这些汇编代码(使用 tag_2 标签缩进)与相应的字节码关联起来后进行注释，能够让我们更好明白他们之间的关联：
```
tag_2：
  // 60 01
  0x1
  // 60 00
  0x0
  // 81
  dup2
  // 90
  swap1
  // 55
  sstore
  // 50
  pop
```
注意上面的汇编代码中``` 0x1``` 实际是 ```push(0x1)``` 的缩写。这条指定将数字 1 推到栈里。

单是看这些代码是无法知道他们究竟是如何工作的。不过别担心， 逐行模拟 EVM 其实很简单。

## 模拟 EVM
  
EVM 是基于栈的虚拟机， 指令会像参数一样在栈上进行操作。让我们先研究下 ```add``` 操作。

假设现在在栈上有两个值：

```
[1 2]
```

当 EVM 执行到 ```add``` 指令时， 它会将栈顶的两个元素连续取出， 计算出结果后再将结果放进栈里。这时候栈空间会变成这样：

```
[3]
```


接下来我们将会用 ```[ ]``` 表示栈：

```
// The empty stack
stack: []
// Stack with three items. The top item is 3. The bottom item is 1.
stack: [3 2 1]
```

用 ```{ }``` 表示合约的存储：

```
// Nothing in storage.
store: {}
// The value 0x1 is stored at the position 0x0.
store: { 0x0 => 0x1 }
```

接下来让我们看看一些实实在在的字节码。我们将会像 EVM 做的那样模拟字节码序列 ```6001600081905550```， 以及打印出每个指令执行后虚拟机的状态：


```
// 60 01: pushes 1 onto stack
0x1
  stack: [0x1]
// 60 00: pushes 0 onto stack
0x0
  stack: [0x0 0x1]
// 81: duplicate the second item on the stack
dup2
  stack: [0x1 0x0 0x1]
// 90: swap the top two items
swap1
  stack: [0x0 0x1 0x1]
// 55: store the value 0x1 at position 0x0
// This instruction consumes the top 2 items
sstore
  stack: [0x1]
  store: { 0x0 => 0x1 }
// 50: pop (throw away the top item)
pop
  stack: []
  store: { 0x0 => 0x1 }

```
  
到最后栈将会为空， 存储空间只会留下一个元素。

值得注意的是， Solidity 会将状态变量 ```uint256 a``` 存放在位置 ```0x0```， 而其他语言完全可以选择将其存储在其他地方。

EVM 对字节码 ```6001600081905550``` 的操作可以用伪代码表示：
```
// a = 1
sstore(0x0, 0x1)
```
仔细观察， 你可能会发现 ```dup2```， ```swap1```， ```pop``` 这些指令是多余的。 汇编代码其实可以变得更加简单：

```
0x1
0x0
sstore
```
你可以尝试模拟上面3个指令， 并且确认自己得到相同的状态机结果。
```
stack: []
store: { 0x0 => 0x1 }
```
## 两个存储变量
接下来我们再增加一个相同类型的存储变量
```
// c2.sol
pragma solidity ^0.4.11;
contract C {
    uint256 a;
    uint256 b;
    function C() {
      a = 1;
      b = 2;
    }
}
```
编译， 然后把注意力放在 ```tag_2``` 标签上： 
```
$ solc --bin --asm c2.sol
// ... more stuff omitted
tag_2:
    /* "c2.sol":99:100  1 */
  0x1
    /* "c2.sol":95:96  a */
  0x0
    /* "c2.sol":95:100  a = 1 */
  dup2
  swap1
  sstore
  pop
    /* "c2.sol":112:113  2 */
  0x2
    /* "c2.sol":108:109  b */
  0x1
    /* "c2.sol":108:113  b = 2 */
  dup2
  swap1
  sstore
  pop
```
对汇编的伪代码表示：
```
// a = 1
sstore(0x0, 0x1)
// b = 2
sstore(0x1, 0x2)
```
我们在这里能学到，两个存储变量的位置是紧连在一起的， a 存储在 ```0x0```， b 存储在 ```0x1```。 

## 存储打包
每个存储槽可以存储 32 个字节的数据。如果一个变量只需要 16 个字节，那么使用 32 个字节来存储它们就显得有些浪费了。 为了优化存储效率， Solidity 会尽可能得通过将两个更小的数据类型打包为一个。

我们将 ```a``` 和 ```b``` 两个变量都改为16字节的类型：
```
pragma solidity ^0.4.11;
contract C {
    uint128 a;
    uint128 b;
    function C() {
      a = 1;
      b = 2;
    }
}
```
然后编译合约：
```
$ solc --bin --asm c3.sol
```
生成的汇编变得更加复杂了：
```
tag_2:
  // a = 1
  0x1
  0x0
  dup1
  0x100
  exp
  dup2
  sload
  dup2
  0xffffffffffffffffffffffffffffffff
  mul
  not
  and
  swap1
  dup4
  0xffffffffffffffffffffffffffffffff
  and
  mul
  or
  swap1
  sstore
  pop
  // b = 2
  0x2
  0x0
  0x10
  0x100
  exp
  dup2
  sload
  dup2
  0xffffffffffffffffffffffffffffffff
  mul
  not
  and
  swap1
  dup4
  0xffffffffffffffffffffffffffffffff
  and
  mul
  or
  swap1
  sstore
  pop
```
上面展示的汇编代码中， 两个变量被存储在同一个地址中```( 0x0 )```， 就像这样：
```
[         b         ][         a         ]
[16 bytes / 128 bits][16 bytes / 128 bits]
```
之所以要这么费劲心思优化存储空间， 是因为到目前为止以太坊花销最大的操作就是在存储上：

- ```sstore```对新位置的首次写入需要花费 20000 个 gas
- ```ssore``` 后续对已存在的地址进行写操作需要花费5000个 gas
- ```sload``` 花费 500 个 gas
- 大多数指令花费 3 到 10 个 gas。 

通过使用相同的存储位置， Solidity 对第二个存储变量只需要花费 5000 个 gas， 而不是 20000， 为我们节省了 15000 个 gas。

## 更多优化
基于同样的思路， EVM也可以将两个 128 字节的数字打包后存储在内存里， 然后只用一个 ```sstore``` 操作就能存储他们， 这样又节省了额外 5000 个 gas

你可以在编译的时候设置 optimize 标识来打开这个优化开关：

```
$ solc --bin --asm --optimize c3.sol
```

生成的汇编代码中，只使用了一个存储槽，并且只执行一次 sstore 操作：

```
tag_2:
    /* "c3.sol":95:96  a */
  0x0
    /* "c3.sol":95:100  a = 1 */
  dup1
  sload
    /* "c3.sol":108:113  b = 2 */
  0x200000000000000000000000000000000
  not(sub(exp(0x2, 0x80), 0x1))
    /* "c3.sol":95:100  a = 1 */
  swap1
  swap2
  and
    /* "c3.sol":99:100  1 */
  0x1
    /* "c3.sol":95:100  a = 1 */
  or
  sub(exp(0x2, 0x80), 0x1)
    /* "c3.sol":108:113  b = 2 */
  and
  or
  swap1
  sstore
```

字节码为：
```
600080547002000000000000000000000000000000006001608060020a03199091166001176001608060020a0316179055
```


将字节码格式化为一行一条指令：

```
// push 0x0
60 00
// dup1
80
// sload
54
// push17 push the the next 17 bytes as a 32 bytes number
70 02 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
/* not(sub(exp(0x2, 0x80), 0x1)) */
// push 0x1
60 01
// push 0x80 (32)
60 80
// push 0x80 (2)
60 02
// exp
0a
// sub
03
// not
19
// swap1
90
// swap2
91
// and
16
// push 0x1
60 01
// or
17
/* sub(exp(0x2, 0x80), 0x1) */
// push 0x1
60 01
// push 0x80
60 80
// push 0x02
60 02
// exp
0a
// sub
03
// and
16
// or
17
// swap1
90
// sstore
55
```
这里有四个魔术变量出现在了汇编代码里：

- 0x1， 占用低位的16个字节

```
// Represented as 0x01 in bytecode

16：32 0x00000000000000000000000000000000
00：16 0x00000000000000000000000000000001
```

- 0x2， 占用高位的16个字节

```
// Represented as 0x200000000000000000000000000000000 in bytecode
16：32 0x00000000000000000000000000000002
00：16 0x00000000000000000000000000000000
```

- ```not ( sub ( exp ( 0x2， 0x80 )， 0x1 ) )```

```
// Bitmask for the upper 16 bytes

16：32 0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF
00：16 0x00000000000000000000000000000000
```


- ```sub ( exp ( 0x2， 0x80 )， 0x1 )```

```
// Bitmask for the lower 16 bytes

16：32 0x00000000000000000000000000000000 
00：16 0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF
```

使用这四个魔术变量对一些比特位进行混淆就能得到期望的结果：

```
16：32 0x00000000000000000000000000000002 
00：16 0x00000000000000000000000000000001
```

最终， 这个 32 个字节的值将被存储在位置 0x0 

## Gas 用量

> 600080547002000000000000000000000000000000006001608060020a03199091166001176001608060020a0316179055

注意 ``` 0x200000000000000000000000000000000 ```被嵌入在了字节码中。但是
编译器也可以选择指令 ```exp ( 0x2， 0x81 )``` 计算出数值， 这种做法可以让字节码序列变得更短。

但事实证明 ```0x200000000000000000000000000000000``` 会比指令 ```exp(0x2， 0x81)``` 更加节省资源。我们可以来计算下 gas 的使用成本：

- 一次交易会有 4 个 gas 被用在每个 0 字节的数据或者代码上
- 一次交易会有 68 个 gas 被用在每个非 0 字节的数据或者代码上


接下来比较下两种方式的 gas 花销：

- 使用字节码 ```0x200000000000000000000000000000000```。 它包含很多个花销更低的 0
 ( 1 * 68 ) + ( 32 * 4 ) = 196



- 使用字节码 ```608160020a``` 。 虽然更短， 但是没有0 
5 * 68 = 340

尽管长度更长， 但包含更多 0 的序列花销会更低 !

## 总结
EVM 编译器并不能精确得优化字节码大小， 速度，或者内存效率。然而， 它却为 gas 的使用做了尽可能的优化，这种另一个层面的优化让以太坊的区块链计算变得更加高效。

我们已经看到 EVM 一些奇特的地方：
- EVM 是一个 256 位的机器。 以 32 字节的大小处理数据是最自然的。
- 持久化存储的成本非常高
- Solidity 编译器为了最小化 gas 的使用做了一些有意义的选择。

本文中设计到我写的一些其他关于 EVM 的文章

- [介绍 EVM 的汇编代码](https：//medium。com/@hayeah/diving-into-the-ethereum-vm-6e8d5d2f3c30)
- [固定长度的数据类型是如何表示的](https：//medium。com/@hayeah/diving-into-the-ethereum-vm-part-2-storage-layout-bc5349cb11b7)
- 动态数据类型是如何被表示的
- 当一个新的合约创建时发生了什么
- 当一个方法被调用时发生了什么
- ABI 如何桥接不同的 EVM 语言
