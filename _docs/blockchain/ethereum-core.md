---
title: 以太坊核心概念
permalink: /docs/blockchain/ethereum-core/
---

## 以太坊虚拟机（EVM）

以太坊虚拟机（EVM）是以太坊中智能合约的运行环境。它不仅被沙箱封装起来，事实上它被完全隔离，也就是说运行在EVM内部的代码不能接触到网络、文件系统或者其它进程。甚至智能合约之间也只有有限的调用。

## 账户（Accounts)

以太坊中有两类账户，它们共用同一个地址空间。外部账户，该类账户被公钥-私钥对控制。合约账户，该类账户被存储在账户中的代码控制。 外部账户的地址是由公钥决定的，合约账户的地址是在创建合约时确定的（这个地址由合约创建者的地址和该地址发出过的交易数量计算得到，地址发出过的交易数量也被称作”nonce”）

两类账户唯一的区别是：合约账户存储了代码，外部账户则没有。

每个账户有一个key-value形式的持久化存储。key，value的长度都是256bit。

另外，每个账户都有一个以太币余额（单位是“Wei”），该账户余额可以通过向它发送带有以太币的交易来改变。。

## 交易（Transactions）

一笔交易是一条消息，从一个账户发送到另一个账户。交易可以包含二进制数据（payload）和以太币。

如果目标账户包含代码（合约账户），该代码和输入数据会被执行。

如果目标账户是零账户（账户地址是0），交易将创建一个新合约。正如上文所讲，这个合约地址不是零地址，而是由合约创建者的地址和该地址发出过的交易数量计算得到。创建合约交易的payload被当作EVM字节码执行。执行的输出做为合约代码被永久存储。这意味着，为了创建一个合约，你不需要向合约发送真正的合约代码，而是发送能够返回真正代码的代码。

对一个合约账户地址，也可以将函数地址放到payload中，这样合约就可以执行指定函数。

## 燃料（Gas）

以太坊上的每笔交易都会被收取一定数量的gas，gas的目的是限制执行交易所需的工作量，同时为执行支付费用。当EVM执行交易时，gas将按照特定规则被逐渐消耗。

gas price（gas价格，以太币计）是由交易创建者设置的，发送账户需要预付的交易费用 = gas price * gas amount。 如果执行结束还有gas剩余，这些gas将被返还给发送账户。

无论执行到什么位置，一旦gas被耗尽（比如降为负值），将会触发一个out-of-gas异常。当前调用帧所做的所有状态修改都将被回滚。


## 存储，主存和栈（Storage, Memory and the Stack）

每个账户有一块持久化内存区域被称为存储。其形式为key-value，key和value的长度均为256比特。在合约里，不能遍历账户的存储。相对于另外两种，存储的读操作相对来说开销较大，修改存储更甚。一个合约只能对它自己的存储进行读写。

第二个内存区被称为主存。合约执行每次消息调用时，都有一块新的，被清除过的主存。主存可以以字节粒度寻址，但是读写粒度为32字节（256比特）。操作主存的开销随着其增长而变大（平方级别）。

EVM不是基于寄存器，而是基于栈的虚拟机。因此所有的计算都在一个被称为栈的区域执行。栈最大有1024个元素，每个元素256比特。对栈的访问只限于其顶端，方式为：允许拷贝最顶端的16个元素中的一个到栈顶，或者是交换栈顶元素和下面16个元素中的一个。所有其他操作都只能取最顶的两个（或一个，或更多，取决于具体的操作）元素，并把结果压在栈顶。当然可以把栈上的元素放到存储或者主存中。但是无法只访问栈上指定深度的那个元素，在那之前必须要把指定深度之上的所有元素都从栈中移除才行。

## 指令集（Instruction Set）

EVM的指令集被刻意保持在最小规模，以尽可能避免可能导致共识问题的错误实现。所有的指令都是针对256比特这个基本的数据类型的操作。具备常用的算术，位，逻辑和比较操作。也可以做到条件和无条件跳转。此外，合约可以访问当前区块的相关属性，比如它的编号和时间戳。

## 消息调用（Message Calls）

合约可以通过消息调用的方式来调用其它合约或者发送以太币到非合约账户。消息调用和交易非常类似，它们都有一个源，一个目标，数据负载，以太币，gas和返回数据。事实上每个交易都可以被认为是一个顶层消息调用，这个消息调用会依次产生更多的消息调用。

一个合约可以决定剩余gas的分配。比如内部消息调用时使用多少gas，或者期望保留多少gas。如果在内部消息调用时发生了out-of-gas异常（或者其他异常），合约将会得到通知，一个错误码被压在栈上。这种情况只是内部消息调用的gas耗尽。在solidity中，这种情况下发起调用的合约默认会触发一个人工异常。这个异常会打印出调用栈。

就像之前说过的，被调用的合约（发起调用的合约也一样）会拥有崭新的主存并能够访问调用的负载。调用负载被存储在一个单独的被称为calldata的区域。调用执行结束后，返回数据将被存放在调用方预先分配好的一块内存中。

调用层数被限制为1024，因此对于更加复杂的操作，我们应该使用循环而不是递归。

## 代码调用和库（Delegatecall / Callcode and Libraries）

存在一种特殊类型的消息调用，被称为callcode。它跟消息调用几乎完全一样，只是加载自目标地址的代码将在发起调用的合约上下文中运行。

这意味着一个合约可以在运行时从另外一个地址动态加载代码。存储，当前地址和余额都指向发起调用的合约，只有代码是从被调用地址获取的。

这使得Solidity可以实现”库“。可复用的库代码可以应用在一个合约的存储上，可以用来实现复杂的数据结构。

## 日志（Logs）

在区块层面，可以用一种特殊的可索引的数据结构来存储数据。这个特性被称为日志，Solidity用它来实现事件。合约创建之后就无法访问日志数据，但是这些数据可以从区块链外高效的访问。因为部分日志数据被存储在布隆过滤器（Bloom filter) 中，我们可以高效并且安全的搜索日志，所以那些没有下载整个区块链的网络节点（轻客户端）也可以找到这些日志。

## 创建（Create）

合约甚至可以通过一个特殊的指令来创建其他合约（不是简单的向零地址发起调用）。创建合约的调用跟普通的消息调用的区别在于，负载数据执行的结果被当作代码，调用者/创建者在栈上得到新合约的地址。

## 自毁（Selfdestruct）

只有在某个地址上的合约执行自毁操作时，合约代码才会从区块链上移除。合约地址上剩余的以太币会发送给指定的目标，然后其存储和代码被移除。 注意，即使一个合约的代码不包含自毁指令，依然可以通过代码调用(callcode)来执行这个操作。