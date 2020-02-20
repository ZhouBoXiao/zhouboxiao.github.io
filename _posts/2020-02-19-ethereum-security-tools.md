---
layout:     post   				    # 使用的布局（不需要改）
title:      Ethereum security tools				# 标题 
subtitle:   以太坊安全工具套件 #副标题
date:       2020-02-19 				# 时间
author:     ZBX 						# 作者
header-img: img/post-bg-2015.jpg 	#这篇文章标题背景图片
catalog: true 						# 是否归档
tags:								#标签
    - Smart Contract
    - 翻译

---

​		要构建一个安全的Ethereum代码库，熟悉要避免的已知错误，对每个新的代码签入进行静态分析，fuzz新特性，并使用symbolic execution验证final product。

**1. Not So Smart Contracts**

​		此[repository](https://github.com/trailofbits/not-so-smart-contracts) 包含常见的Ethereum智能合约漏洞示例，包括实际代码。回顾一下这个列表，确保你对可能出现的问题非常熟悉。repository包含针对每一类漏洞的子目录，如[integer overflow](https://github.com/trailofbits/not-so-smart-contracts/tree/master/integer_overflow), [reentrancy](https://github.com/trailofbits/not-so-smart-contracts/tree/master/reentrancy), and [unprotected functions](https://github.com/trailofbits/not-so-smart-contracts/tree/master/unprotected_function).。每个子目录都包含自己的readme和real-world中的脆弱合约示例。在适当的情况下，还提供利用漏洞的合约。

**2. Slither**

[		Slither](https://github.com/trailofbits/slither) 结合了一组关于可靠性的静态分析，这些分析可以检测常见的错误，如reentrancy、constructors、method access等方面的错误。在每一个新的代码签入上，run Slither as you develop。我们不断地合并我们在审计中发现的新的、独特的bug类型。

Running Slither is simple: 

```
$ slither.py contract.sol
```

Slither will then output the vulnerabilities it finds in the contract.

**3. Echidna**

[		Echidna](https://github.com/trailofbits/echidna)将下一代智能fuzz技术应用于EVM字节码。完成新特性后，为代码编写Echidna测试。它提供了简单、高覆盖率的单元测试来发现安全漏洞。在你的应用程序有80%以上的覆盖率之前，不要认为它是完整的。

Using Echidna is simple:

1. Add some Echidna tests to your existing code (like in this [example](https://github.com/trailofbits/echidna/blob/master/solidity/cli.sol)),
2. Run ./echidna-test contract.sol, and
3. See if your invariants hold.

```
ethsec@cb72cd7112e0:~/echidna/examples/solidity/truffle/metacoin$ echidna-test . TEST INFO:CryticCompile:'npx truffle compile --all' running (use --truffle-version truffle@x.x.x to use specific version) INFO:CryticCompile: Compiling your contracts... =========================== > Compiling ./contracts/ConvertLib.sol > Compiling ./contracts/MetaCoin.sol > Compiling ./contracts/MetaCoinEchidna.sol > Compiling ./contracts/Migrations.sol > Artifacts written to /home/ethsec/echidna/examples/solidity/truffle/metacoin/build/contracts > Compiled successfully using:   - solc: 0.5.8+commit.23d335f2.Emscripten.clang  ERROR:CryticCompile:- Fetching solc version list from solc-bin. Attempt #1 - Downloading compiler. Attempt #1 Analyzing contract: /home/ethsec/echidna/examples/solidity/truffle/metacoin/contracts/MetaCoinEchidna.sol:TEST echidna_convert: failed!💥    Call sequence:    mint(76225693455075590054801325349986579957993867907618604313176432338640747599638) Gas price: 0x5ed524149 Time delay: 0x1762a Block delay: 0xbfdd  Seed: -1812525598806959557

ethsec@cb72cd7112e0:~/echidna$ echidna-test examples/solidity/basic/flags.sol Analyzing contract: /home/ethsec/echidna/examples/solidity/basic/flags.sol:Test echidna_sometimesfalse: failed!💥    Call sequence:    set0(0)    set1(0) echidna_alwaystrue: passed! 🎉 echidna_revert_always: passed! 🎉 Seed: -1980352839675882049
```

**4. Manticore**

​		[Manticore](https://github.com/trailofbits/manticore)使用符号执行来模拟针对EVM字节码的复杂的muti-contractg和muti-transcation攻击。一旦你的应用程序是功能性的，写Manticore测试，以发现隐藏的，意想不到的，或危险的状态，它可以进入。Manticore枚举合约的执行状态并验证关键功能。

If your contract doesn’t require initialization parameters, then you can use the command line to easily explore all the possible executions of your smart contract as an attacker or the contract owner:

manticore contract.sol --contract ContractName --txaccount [attacker|owner]

​		Manticore将生成所有可到达状态(包括断言失败和恢复)的列表，以及导致这些状态的输入。它还会自动标记某些类型的问题，比如整数溢出和未初始化内存的使用。

Using the Manticore API to review more advanced contracts is simple:

1. [Initialize](https://github.com/trailofbits/manticore/blob/master/examples/evm/complete.py#L26) your contract with the proper values
2. [Define](https://github.com/trailofbits/manticore/blob/master/examples/evm/complete.py#L32-L37) symbolic transactions to explore potential states
3. [Review](https://github.com/trailofbits/manticore/blob/master/examples/evm/complete.py#L41-L43) the list of resulting transactions for undesirable states

## Reversing Tools

​		一旦你开发了你的智能合同，或者你想看看别人的代码，你将需要使用我们的逆向工具。将二进制合同加载到Ethersplay或IDA-EVM中。对于指令集引用，请使用我们的EVM操作码数据库。如果你想做更复杂的分析，使用Rattle。

**1. EVM Opcode Database**

​		无论您是在Remix调试器中单步调试代码，还是对二进制契约进行反向工程，您都可能需要查看EVM指令的详细信息。这个[reference](https://github.com/trailofbits/evm-opcodes)包含一个完整而简明的EVM操作码及其实现细节列表。我们认为，与滚动黄色的纸张、阅读Go/Rust源代码或检查StackOverflow文章中的评论相比，这是一个非常节省时间的方法。

**2. Ethersplay**

[		Ethersplay](https://github.com/trailofbits/ethersplay)是一个图形化的EVM反汇编程序，能够进行方法恢复、动态跳转计算、源代码匹配和 binary diffing。使用Ethersplay调查和调试已编译的契约或已部署到区块链的契约。

​		Ethersplay以ascii十六进制编码或原始二进制格式的EVM字节码作为输入。每个例子都是 [test.evm](https://github.com/trailofbits/ethersplay/blob/master/examples/test.evm) and [test.bytecode](https://github.com/trailofbits/ethersplay/blob/master/examples/test.bytecode)。打开test.evm文件中的二进制Ninja，它会自动分析它，识别功能，并生成一个控制流图。Ethersplay包括两个帮助忍者的二进制插件。EVM源代码将把契约源与EVM字节码关联起来。EVM Manticore Highlight集成了Manticore和Ethersplay，图形化地突出显示来自Manticore输出的代码覆盖率信息。

**3. IDA-EVM**

[		IDA-EVM](https://github.com/trailofbits/ida-evm) is a graphical EVM disassembler for IDA Pro capable of function recovery, dynamic jump computation, applying library signatures, and binary diffing using BinDiff. 

​		IDA-EVM allows you to analyze and reverse engineer smart contracts without source. To use it, follow the installation instructions in the readme, then open a .evm or .bytecode file in IDA.

**4. Rattle**

​		[Rattle](https://github.com/trailofbits/rattle)是一个EVM静态分析器，它直接分析EVM字节码的漏洞。它通过分解和恢复EVM控制流图，并将操作提升到一个称为EVM::SSA的静态赋值(SSA)表单来实现这一点。EVM::SSA优化了所有的push、pop、dups和swap，通常将指令数减少了75%。最终，Rattle 将支持存储、内存和参数恢复，以及类似于Slither中实现的静态安全检查。

To use Rattle, supply it runtime bytecode from solc or extracted directly from the blockchain:

```
$ ./rattle -i path/to/input.evm
```



原文链接：https://blog.trailofbits.com/2018/03/23/use-our-suite-of-ethereum-security-tools/

