---
layout:     post   				    # 使用的布局（不需要改）
title:      EVM Interpreter			# 标题 
subtitle:   以太坊虚拟机的解释器部分源码分析 #副标题
date:       2020-02-17 				# 时间
author:     ZBX 						# 作者
header-img: img/post-bg-2015.jpg 	#这篇文章标题背景图片
catalog: true 						# 是否归档
tags:								#标签
    - Ethereum
---

​		在以太坊虚拟机中，解释器扮演着重要的角色，想要弄清楚解释器的原理，必须从虚拟机的源码层面上进行分析。

​		从instruction.go文件中的指令解释入手，因为指令很多，只列举几个例子：

```go
func opPc(pc *uint64, evm *EVM, contract *Contract, memory *Memory, stack *Stack) ([]byte, error) {
    	stack.push(evm.interpreter.intPool.get().SetUint64(*pc))
    	return nil, nil
}

func opMsize(pc *uint64, evm *EVM, contract *Contract, memory *Memory, stack *Stack) ([]byte, error) {
    	stack.push(evm.interpreter.intPool.get().SetInt64(int64(memory.Len())))
    	return nil, nil
}
```



#### interpreter.go 解释器数据结构

```go
// Config are the configuration options for the Interpreter
type Config struct {
    	// Debug enabled debugging Interpreter options
    	Debug bool
    	// EnableJit enabled the JIT VM
    	EnableJit bool
    	// ForceJit forces the JIT VM
    	ForceJit bool
    	// Tracer is the op code logger
	Tracer Tracer
	// NoRecursion disabled Interpreter call, callcode,
	// delegate call and create.
	NoRecursion bool
	// Disable gas metering
	DisableGasMetering bool
	// Enable recording of SHA3/keccak preimages
	EnablePreimageRecording bool
	// JumpTable contains the EVM instruction table. This
	// may be left uninitialised and will be set to the default
	// table.
	JumpTable [256]operation
}

// Interpreter is used to run Ethereum based contracts and will utilise the
// passed evmironment to query external sources for state information.
// The Interpreter will run the byte code VM or JIT VM based on the passed
// configuration.
type Interpreter struct {
	evm      *EVM
	cfg      Config
	gasTable params.GasTable   // 标识了很多操作的Gas价格
	intPool  *intPool

	readOnly   bool   // Whether to throw on stateful modifications
	returnData []byte // Last CALL's return data for subsequent reuse 最后一个函数的返回值
}
```

#### 构造函数

```go
// NewInterpreter returns a new instance of the Interpreter.
func NewInterpreter(evm *EVM, cfg Config) *Interpreter {
	// We use the STOP instruction whether to see
	// the jump table was initialised. If it was not
	// we'll set the default jump table.
	// 用一个STOP指令测试JumpTable是否已经被初始化了, 如果没有被初始化,那么设置为默认值
	if !cfg.JumpTable[STOP].valid { 
		switch {
		case evm.ChainConfig().IsByzantium(evm.BlockNumber):
			cfg.JumpTable = byzantiumInstructionSet
		case evm.ChainConfig().IsHomestead(evm.BlockNumber):
			cfg.JumpTable = homesteadInstructionSet
		default:
			cfg.JumpTable = frontierInstructionSet
		}
	}

	return &Interpreter{
		evm:      evm,
		cfg:      cfg,
		gasTable: evm.ChainConfig().GasTable(evm.BlockNumber),
		intPool:  newIntPool(),
	}
}
```

解释器一共就两个方法enforceRestrictions方法和Run方法

```go
func (in *Interpreter) enforceRestrictions(op OpCode, operation operation, stack *Stack) error {
	if in.evm.chainRules.IsByzantium {
		if in.readOnly {
			// If the interpreter is operating in readonly mode, make sure no
			// state-modifying operation is performed. The 3rd stack item
			// for a call operation is the value. Transferring value from one
			// account to the others means the state is modified and should also
			// return with an error.
			if operation.writes || (op == CALL && stack.Back(2).BitLen() > 0) {
				return errWriteProtection
			}
		}
	}
	return nil
}

// Run loops and evaluates the contract's code with the given input data and returns
// the return byte-slice and an error if one occurred.
// 用给定的入参循环执行合约的代码，并返回返回的字节片段，如果发生错误则返回错误。
// It's important to note that any errors returned by the interpreter should be
// considered a revert-and-consume-all-gas operation. No error specific checks
// should be handled to reduce complexity and errors further down the in.
// 重要的是要注意，解释器返回的任何错误都会消耗全部gas。 为了减少复杂性,没有特别的错误处理流程。
func (in *Interpreter) Run(snapshot int, contract *Contract, input []byte) (ret []byte, err error) {
	// Increment the call depth which is restricted to 1024
	in.evm.depth++
	defer func() { in.evm.depth-- }()

	// Reset the previous call's return data. It's unimportant to preserve the old buffer
	// as every returning call will return new data anyway.
	in.returnData = nil

	// Don't bother with the execution if there's no code.
	if len(contract.Code) == 0 {
		return nil, nil
	}

	codehash := contract.CodeHash // codehash is used when doing jump dest caching
	if codehash == (common.Hash{}) {
		codehash = crypto.Keccak256Hash(contract.Code)
	}

	var (
		op    OpCode        // current opcode
		mem   = NewMemory() // bound memory
		stack = newstack()  // local stack
		// For optimisation reason we're using uint64 as the program counter.
		// It's theoretically possible to go above 2^64. The YP defines the PC
		// to be uint256. Practically much less so feasible.
		pc   = uint64(0) // program counter
		cost uint64
		// copies used by tracer
		stackCopy = newstack() // stackCopy needed for Tracer since stack is mutated by 63/64 gas rule 
		pcCopy uint64 // needed for the deferred Tracer
		gasCopy uint64 // for Tracer to log gas remaining before execution
		logged bool // deferred Tracer should ignore already logged steps
	)
	contract.Input = input

	defer func() {
		if err != nil && !logged && in.cfg.Debug {
			in.cfg.Tracer.CaptureState(in.evm, pcCopy, op, gasCopy, cost, mem, stackCopy, contract, in.evm.depth, err)
		}
	}()

	// The Interpreter main run loop (contextual). This loop runs until either an
	// explicit STOP, RETURN or SELFDESTRUCT is executed, an error occurred during
	// the execution of one of the operations or until the done flag is set by the
	// parent context.
	// 解释器的主要循环， 直到遇到STOP，RETURN，SELFDESTRUCT指令被执行，或者是遇到任意错误，或者说done 标志被父context设置。
	for atomic.LoadInt32(&in.evm.abort) == 0 {
		// Get the memory location of pc
		// 得到下一条指令的内存地址
		op = contract.GetOp(pc)

		if in.cfg.Debug {
			logged = false
			pcCopy = uint64(pc)
			gasCopy = uint64(contract.Gas)
			stackCopy = newstack()
			for _, val := range stack.data {
				stackCopy.push(val)
			}
		}

		// get the operation from the jump table matching the opcode
		// 通过JumpTable拿到对应的operation
		operation := in.cfg.JumpTable[op]
		// 这里检查了只读模式下面不能执行writes指令
		// staticCall的情况下会设置为readonly模式
		if err := in.enforceRestrictions(op, operation, stack); err != nil {
			return nil, err
		}

		// if the op is invalid abort the process and return an error
		if !operation.valid { //检查指令是否非法
			return nil, fmt.Errorf("invalid opcode 0x%x", int(op))
		}

		// validate the stack and make sure there enough stack items available
		// to perform the operation
		// 检查是否有足够的堆栈空间。 包括入栈和出栈
		if err := operation.validateStack(stack); err != nil {
			return nil, err
		}

		var memorySize uint64
		// calculate the new memory size and expand the memory to fit
		// the operation
		if operation.memorySize != nil { // 计算内存使用量，需要收费
			memSize, overflow := bigUint64(operation.memorySize(stack))
			if overflow {
				return nil, errGasUintOverflow
			}
			// memory is expanded in words of 32 bytes. Gas
			// is also calculated in words.
			if memorySize, overflow = math.SafeMul(toWordSize(memSize), 32); overflow {
				return nil, errGasUintOverflow
			}
		}

		if !in.cfg.DisableGasMetering { //这个参数在本地模拟执行的时候比较有用，可以不消耗或者检查GAS执行交易并得到返回结果
			// consume the gas and return an error if not enough gas is available.
			// cost is explicitly set so that the capture state defer method cas get the proper cost
			// 计算gas的Cost 并使用，如果不够，就返回OutOfGas错误。
			cost, err = operation.gasCost(in.gasTable, in.evm, contract, stack, mem, memorySize)
			if err != nil || !contract.UseGas(cost) {
				return nil, ErrOutOfGas
			}
		}
		if memorySize > 0 { //扩大内存范围
			mem.Resize(memorySize)
		}

		if in.cfg.Debug {
			in.cfg.Tracer.CaptureState(in.evm, pc, op, gasCopy, cost, mem, stackCopy, contract, in.evm.depth, err)
			logged = true
		}

		// execute the operation
		// 执行命令
		res, err := operation.execute(&pc, in.evm, contract, mem, stack)
		// verifyPool is a build flag. Pool verification makes sure the integrity
		// of the integer pool by comparing values to a default value.
		if verifyPool {
			verifyIntegerPool(in.intPool)
		}
		// if the operation clears the return data (e.g. it has returning data)
		// set the last return to the result of the operation.
		if operation.returns { //如果有返回值，那么就设置返回值。 注意只有最后一个返回有效果。
			in.returnData = res
		}

		switch {
		case err != nil:
			return nil, err
		case operation.reverts:
			return res, errExecutionReverted
		case operation.halts:
			return res, nil
		case !operation.jumps:
			pc++
		}
	}
	return nil, nil
}
```

---

### 下面用一个程序作为例子

```javascript
pragma solidity >=0.4.22 <0.6.0;
contract Ballot {

    struct Voter {
        uint weight;
        bool voted;
        uint8 vote;
        address delegate;
    }
    struct Proposal {
        uint voteCount;
    }

    address chairperson;
    mapping(address => Voter) voters;
    Proposal[] proposals;

    /// Create a new ballot with $(_numProposals) different proposals.
    constructor(uint8 _numProposals) public {
        chairperson = msg.sender;
        voters[chairperson].weight = 1;
        proposals.length = _numProposals;
    }

    /// Give $(toVoter) the right to vote on this ballot.
    /// May only be called by $(chairperson).
    function giveRightToVote(address toVoter) public {
        if (msg.sender != chairperson || voters[toVoter].voted) return;
        voters[toVoter].weight = 1;
    }

    /// Delegate your vote to the voter $(to).
    function delegate(address to) public {
        Voter storage sender = voters[msg.sender]; // assigns reference
        if (sender.voted) return;
        while (voters[to].delegate != address(0) && voters[to].delegate != msg.sender)
            to = voters[to].delegate;
        if (to == msg.sender) return;
        sender.voted = true;
        sender.delegate = to;
        Voter storage delegateTo = voters[to];
        if (delegateTo.voted)
            proposals[delegateTo.vote].voteCount += sender.weight;
        else
            delegateTo.weight += sender.weight;
    }

    /// Give a single vote to proposal $(toProposal).
    function vote(uint8 toProposal) public {
        Voter storage sender = voters[msg.sender];
        if (sender.voted || toProposal >= proposals.length) return;
        sender.voted = true;
        sender.vote = toProposal;
        proposals[toProposal].voteCount += sender.weight;
    }

    function winningProposal() public view returns (uint8 _winningProposal) {
        uint256 winningVoteCount = 0;
        for (uint8 prop = 0; prop < proposals.length; prop++)
            if (proposals[prop].voteCount > winningVoteCount) {
                winningVoteCount = proposals[prop].voteCount;
                _winningProposal = prop;
            }
    }
}

```

**RUNTIME BYTECODE**

```json
{
	"linkReferences": {},
	"object": "6080604052348015610010576000...",
	"sourceMap": "33:2130:0:-;;;;8:9:-1;5:2;;..."
}
```

**ABI**（Application Binary Interface）

```json
[
	{
		"constant": false,
		"inputs": [
			{
				"internalType": "address",
				"name": "to",
				"type": "address"
			}
		],
		"name": "delegate",
		"outputs": [],
		"payable": false,
		"stateMutability": "nonpayable",
		"type": "function"
	},
	{
		"constant": true,
		"inputs": [],
		"name": "winningProposal",
		"outputs": [
			{
				"internalType": "uint8",
				"name": "_winningProposal",
				"type": "uint8"
			}
		],
		"payable": false,
		"stateMutability": "view",
		"type": "function"
	},
	{
		"constant": false,
		"inputs": [
			{
				"internalType": "address",
				"name": "toVoter",
				"type": "address"
			}
		],
		"name": "giveRightToVote",
		"outputs": [],
		"payable": false,
		"stateMutability": "nonpayable",
		"type": "function"
	},
	{
		"constant": false,
		"inputs": [
			{
				"internalType": "uint8",
				"name": "toProposal",
				"type": "uint8"
			}
		],
		"name": "vote",
		"outputs": [],
		"payable": false,
		"stateMutability": "nonpayable",
		"type": "function"
	},
	{
		"inputs": [
			{
				"internalType": "uint8",
				"name": "_numProposals",
				"type": "uint8"
			}
		],
		"payable": false,
		"stateMutability": "nonpayable",
		"type": "constructor"
	}
]
```

**ASSEMBLY**

```
.code
  PUSH 80			contract Ballot {\n\n    struc...
  PUSH 40			contract Ballot {\n\n    struc...
  MSTORE 			contract Ballot {\n\n    struc...
  CALLVALUE 			constructor(uint8 _numProposal...
  DUP1 			olidity >
  ISZERO 			a 
  PUSH [tag] 1			a 
  JUMPI 			a 
  PUSH 0			0
  DUP1 			.
  REVERT 			4.22 <0.6.0;
tag 1			a 
  JUMPDEST 			a 
  POP 			constructor(uint8 _numProposal...
  PUSH 40			constructor(uint8 _numProposal...
  MLOAD 			constructor(uint8 _numProposal...
  PUSHSIZE 			constructor(uint8 _numProposal...
  CODESIZE 			constructor(uint8 _numProposal...
  SUB 			constructor(uint8 _numProposal...
  DUP1 			constructor(uint8 _numProposal...
  PUSHSIZE 			constructor(uint8 _numProposal...
  DUP4 			constructor(uint8 _numProposal...
  CODECOPY 			constructor(uint8 _numProposal...
  DUP2 			constructor(uint8 _numProposal...
  DUP2 			constructor(uint8 _numProposal...
  ...
  ...
  ...
```



solidity中`memory／storage`关键字，分别是对应着值类型、引用类型，`storage` 变量是指永久存储在区块链中的变量。 `memory `变量则是临时的，当对某合约调用完成时，内存型变量即被移除。

`storage `指令对应的操作是：

```go
func opSload(pc *uint64, interpreter *EVMInterpreter, contract *Contract, memory *Memory, stack *Stack) ([]byte, error) {
    loc := stack.peek() //不删除栈顶的值
    val := interpreter.evm.StateDB.GetState(contract.Address(), common.BigToHash(loc))
    loc.SetBytes(val.Bytes())  //把结果放到stack中
    return nil, nil
}

func opSstore(pc *uint64, interpreter *EVMInterpreter, contract *Contract, memory *Memory, stack *Stack) ([]byte, error) {
     loc := common.BigToHash(stack.pop())
    val := stack.pop()
    interpreter.evm.StateDB.SetState(contract.Address(), loc, common.BigToHash(val))

    interpreter.intPool.put(val)
    return nil, nil
}
```



其中StateDB 的结构定义如下：

```go
type StateDB struct {
    db   Database
    trie Trie
    // This map holds 'live' objects, which will get modified while processing a state transition.
    stateObjects      map[common.Address]*stateObject
    stateObjectsDirty map[common.Address]struct{}

    // DB error.
    // State objects are used by the consensus core and VM which are
    // unable to deal with database-level errors. Any error that occurs
    // during a database read is memoized here and will eventually be returned
    // by StateDB.Commit.
    dbErr error

    // The refund counter, also used by state transitioning.
    refund uint64

    thash, bhash common.Hash
    txIndex      int
    logs         map[common.Hash][]*types.Log
    logSize      uint

    preimages map[common.Hash][]byte

    // Journal of state modifications. This is the backbone of
    // Snapshot and RevertToSnapshot.
    journal        *journal
    validRevisions []revision
    nextRevisionId int

    // Measurements gathered during execution for debugging purposes
    AccountReads   time.Duration
    AccountHashes  time.Duration
    AccountUpdates time.Duration
    AccountCommits time.Duration
    StorageReads   time.Duration
    StorageHashes  time.Duration
    StorageUpdates time.Duration
    StorageCommits time.Duration
}
```

stateDB中的setState和getState方法如下：

```go
// GetState retrieves a value from the given account's storage trie.
func (self *StateDB) GetState(addr common.Address, hash common.Hash) common.Hash {
    stateObject := self.getStateObject(addr)
    if stateObject != nil {
        return stateObject.GetState(self.db, hash)
    }
    return common.Hash{}
}

func (self *StateDB) SetState(addr common.Address, key, value common.Hash) {
    stateObject := self.GetOrNewStateObject(addr)
    if stateObject != nil {
        stateObject.SetState(self.db, key, value)
    }
}
```

其中stateObject结构

```go
type stateObject struct {
    address  common.Address
    addrHash common.Hash // hash of ethereum address of the account
    data     Account
    db       *StateDB

    // DB error.
    // State objects are used by the consensus core and VM which are
    // unable to deal with database-level errors. Any error that occurs
    // during a database read is memoized here and will eventually be returned
    // by StateDB.Commit.
    dbErr error

    // Write caches.
    trie Trie // storage trie, which becomes non-nil on first access
    code Code // contract bytecode, which gets set when code is loaded

    originStorage Storage // Storage cache of original entries to dedup rewrites
    dirtyStorage  Storage // Storage entries that need to be flushed to disk

    // Cache flags.
    // When an object is marked suicided it will be delete from the trie
    // during the "update" phase of the state transition.
    dirtyCode bool // true if the code was updated
    suicided  bool
    deleted   bool
}
```



查看StateDB和stateObject的定义可以发现，这两种类型内部各有一个Trie，StateDB里的Trie以账户地址为key，存储经过RLP编码后的stateObject。stateObject里的Trie也被称为storage trie，存储的是智能合约执行后修改的变量值。

下面是stateObject中GetState和SetState方法：

```go
// GetState retrieves a value from the account storage trie.
func (s *stateObject) GetState(db Database, key common.Hash) common.Hash {
    // If we have a dirty value for this state entry, return it
    value, dirty := s.dirtyStorage[key]
    if dirty {
        return value
    }
    // Otherwise return the entry's original value
    return s.GetCommittedState(db, key)
}

// GetCommittedState 从committed帐户的storage trie检索一个值
func (s *stateObject) GetCommittedState(db Database, key common.Hash) common.Hash {
    // If we have the original value cached, return that
    value, cached := s.originStorage[key]
    if cached {
        return value
    }
    // Track the amount of time wasted on reading the storge trie
    if metrics.EnabledExpensive {
        defer func(start time.Time) { s.db.StorageReads += time.Since(start) }(time.Now())
    }
    // 否则从数据库加载值
    enc, err := s.getTrie(db).TryGet(key[:])
    if err != nil {
        s.setError(err)
        return common.Hash{}
    }
    if len(enc) > 0 {
        _, content, _, err := rlp.Split(enc)
        if err != nil {
            s.setError(err)
        }
        value.SetBytes(content)
    }
    s.originStorage[key] = value
    return value
}
```

```go
// SetState updates a value in account storage.
func (s *stateObject) SetState(db Database, key, value common.Hash) {
    // If the new value is the same as old, don't set
    prev := s.GetState(db, key)
    if prev == value {
        return
    }
    // New value is different, update and journal the change
    s.db.journal.append(storageChange{
        account:  &s.address,
        key:      key,
        prevalue: prev,
    })
    s.setState(key, value)
}
func (s *stateObject) setState(key, value common.Hash) {
    s.dirtyStorage[key] = value
}
```



StateDB中存储了很多stateObject，而每一个stateObject则代表了一个以太坊账户，包含了账户的地址、余额、nonce、合约代码hash等状态信息。所有账户的当前状态在以太坊中被称为“世界状态”，在每次挖出或者接收到新区块时需要更新世界状态。

为了能够快速检索和更新账户状态，StateDB采用了两级缓存机制：

-   第一级缓存以map的形式存储stateObject
-   第二级缓存以MPT的形式存储
-   第三级就是LevelDB上的持久化存储

当上一级缓存中没有所需的数据时，会从下一级缓存或者数据库中进行加载。

一共封装了3个包：state，trie，ethdb。如果按接口类型来分，主要分为Trie和Database两种接口。Trie接口主要用于操作内存中的MPT，而Database接口主要用于操作LevelDB，做持久化存储。StateDB中同时包含了这两种接口。