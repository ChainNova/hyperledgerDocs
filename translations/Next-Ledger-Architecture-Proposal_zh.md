---
layout: page
title: "[翻译]Next-Ledger-Architecture-Proposal"
description: ""
---
{% include JB/setup %}

原文：<https://github.com/hyperledger/fabric/blob/master/proposals/r1/Next-Ledger-Architecture-Proposal.md>


翻译：[梧桐树](https://wutongtree.com)

---


**草案** / **进行中**

该文档是基于社区反馈的未来ledger架构的一个提案。 所有的输入都是社区努力的目标。

##### 目录  

[目的](#motivation)  
[API](#api)  
[时间点查询](#pointintime)  
[查询语言](#querylanguage)


## <a name="motivation"></a>目的

探索新ledger架构的动机来自于社区反馈。现存的ledger可以支持一些（但不是所有）下面的需求，我们要探索一种新的ledger来满足已经了解的所有需求。 根据在很多社区（Slack、Github等）及面对面的讨论，很显然，大家强烈希望支持以下需求：

1. 时间点查询 - 在上一区块查询chaincode state，容易跟踪**没有**重放交易的谱系
2. 像SQL一样的的查询语言
3. 隐私 - 完整的ledger可能不会存在于所有的committers
4. 加密的安全ledger - 数据完整性不参照其他节点
5. 支持像PBFT那样提供即时终结的共识算法
6. 支持像PoW、PoET那样需要随机收敛的共识算法
7. 裁剪 - 根据需要删除旧的交易数据
8. 支持背书从共识中分离，详情见 [下一代共识架构提案](http://wutongtree.github.io/translations/Next-Consensus-Architecture-Proposal_zh)。 这意味着，某些peer在没有执行交易或查看chaincode逻辑的情况下想他们的ledger申请背书结果。
9. API / Enginer 分离。按需插入不同的存储引擎。


## <a name="api"></a>API

以Go伪代码写成的建议API

```
package ledger

import "github.com/hyperledger/fabric/protos"

// Encryptor is an interface that a ledger implementation can use for Encrypt/Decrypt the chaincode state
type Encryptor interface {
	Encrypt([]byte) []byte
	Decrypt([]byte) []byte
}

// PeerMgmt is an interface that a ledger implementation expects from peer implementation
// 
type PeerMgmt interface {
	// IsPeerEndorserFor returns 'true' if the peer is endorser for given chaincodeID
	IsPeerEndorserFor(chaincodeID string) bool

	// ListEndorsingChaincodes return the chaincodeIDs for which the peer acts as one of the endorsers
	ListEndorsingChaincodes() []string

	// GetEncryptor returns the Encryptor for the given chaincodeID
	GetEncryptor(chaincodeID string) (Encryptor, error)
}

// In the case of a confidential chaincode, the simulation results from ledger are expected to be encrypted using the 'Encryptor' corresponding to the chaincode.
// Similarly, the blocks returned by the GetBlock(s) method of the ledger are expected to have the state updates in the encrypted form.
// However, internally, the ledger can maintain the latest and historical state for the chaincodes for which the peer is one of the endorsers - in plain text form.
// TODO - Is this assumption correct?

// General purpose interface for forcing a data element to be serializable/de-serializable
type DataHolder interface {
	GetData() interface{}
	GetBytes() []byte
	DecodeBytes(b []byte) interface{}
}

type SimulationResults interface {
	DataHolder
}

type QueryResult interface {
	DataHolder
}

type BlockHeader struct {
}

type PrunePolicy interface {
}

type BlockRangePrunePolicy struct {
	FirstBlockHash string
	LastBlockHash  string
}

// QueryExecutor executes the queries
// Get* methods are for supporting KV-based data model. ExecuteQuery method is for supporting a rich datamodel and query support
//
// ExecuteQuery method in the case of a rich data model is expected to support queries on
// latest state, historical state and on the intersection of state and transactions
type QueryExecutor interface {
	GetState(key string) ([]byte, error)
	GetStateRangeScanIterator(startKey string, endKey string) (ResultsIterator, error)
	GetStateMultipleKeys(keys []string) ([][]byte, error)
	GetTransactionsForKey(key string) (ResultsIterator, error)

	ExecuteQuery(query string) (ResultsIterator, error)
}

// TxSimulator simulates a transaction on a consistent snapshot of the as recent state as possible
type TxSimulator interface {
	QueryExecutor
	StartNewTx()

	// KV data model
	SetState(key string, value []byte)
	DeleteState(key string)
	SetStateMultipleKeys(kvs map[string][]byte)

	// for supporting rich data model (see comments on QueryExecutor above)
	ExecuteUpdate(query string)

	// This can be a large payload
	CopyState(sourceChaincodeID string) error

	// GetTxSimulationResults encapsulates the results of the transaction simulation.
	// This should contain enough detail for
	// - The update in the chaincode state that would be caused if the transaction is to be committed
	// - The environment in which the transaction is executed so as to be able to decide the validity of the enviroment
	//   (at a later time on a different peer) during committing the transactions
	// Different ledger implementation (or configurations of a single implementation) may want to represent the above two pieces
	// of information in different way in order to support different data-models or optimize the information representations.
	// TODO detailed illustration of a couple of representations.
	GetTxSimulationResults() SimulationResults
	HasConflicts() bool
	Clear()
}

type ResultsIterator interface {
	// Next moves to next key-value. Returns true if next key-value exists
	Next() bool
	// GetKeyValue returns next key-value
	GetResult() QueryResult
	// Close releases resources occupied by the iterator
	Close()
}

// Ledger represents the 'final ledger'. In addition to implement the methods inherited from the BasicLedger,
// it provides the handle to objects for querying the chaincode state and executing chaincode transactions.
type ValidatedLedger interface {
	Ledger
	// NewTxSimulator gives handle to a transaction simulator for given chaincode and given fork (represented by blockHash)
	// A client can obtain more than one 'TxSimulator's for parallel execution. Any synchronization should be performed at the
	// implementation level if required
	NewTxSimulator(chaincodeID string, blockHash string) (TxSimulator, error)

	// NewQueryExecuter gives handle to a query executer for given chaincode and given fork (represented by blockHash)
	// A client can obtain more than one 'QueryExecutor's for parallel execution. Any synchronization should be performed at the
	// implementation level if required
	NewQueryExecuter(chaincodeID string, blockHash string) (QueryExecutor, error)

	// CheckpointPerformed is expected to be invoked by the consensus algorithm when it completes a checkpoint across peers
	// On the invoke of this method, the block in the 'RawLedger' between the 'corresponding to the block currently checkpointed' and
	// 'corresponding to the block checkpointed last time' can be pruned.
	// (Pruning the raw blocks would need an additional time based factor as well, if forks are to be supported in the raw ledger.)
	// (Does raw ledger in the case of a consensus that allow forks (e.g., PoW) make sense at all? Or practically, these consensus
	// would always produce the final blocks that contains validated transactions).
	CheckpointPerformed(blockHash string)

	RemoveInvalidTransactions(block *protos.Block) (*protos.Block, error)
}

// RawLedger implements methods required by 'raw ledger'
// CommitBlock() of RawLedger is expected to be invoked by the consensus algorithm when a new block is constructed.
// Upon receiving the new block, it is expected to be 'processed' by the ledger - the processing includes -
// preserving the raw block, validating each transaction in the block, discarding the invalid transactions,
// preparing the final block with the rmaining (i.e. valid) transactions and committing the final block (including updating the state).
// The raw block should not be deleted as yet - until the corresponding 'final block' is included in one of the following checkpoint performed by the consensus.
type RawLedger interface {
	Ledger
}

//Ledger captures the methods that are common across the 'raw ledger' and the 'final ledger'
type Ledger interface {
	//GetTopBlockHashes returns the hashes of the top most block in each fork.
	GetTopBlockHashes() []string
	//CommitBlock adds a new block
	CommitBlock(block *protos.Block) error
	//GetTransactionByID retrieves a transaction by id
	GetTransactionByID(txID string) (*protos.Transaction, error)
	//GetBlockChain returns an instance of chain that starts at the
	GetBlockChain(topBlockHash string) (BlockChain, error)
	//Prune prunes the blocks/transactions that satisfy the given policy
	Prune(policy PrunePolicy) error
}

//BlockChain represents an instance of a block chain. In the case of a consensus algorithm that could cause a fork, an instance of BlockChain
// represent one of the forks (i.e., one of the chains starting from the genesis block to the one of the top most blocks)
type BlockChain interface {
	GetTopBlockHash() string
	GetBlockchainInfo() (*protos.BlockchainInfo, error)
	GetBlockHeaders(startingBlockHash, endingBlockHash string) []*BlockHeader
	GetBlocks(startingBlockHash, endingBlockHash string) []*protos.Block
	GetBlockByNumber(blockNumber uint64) *protos.Block
	GetBlocksByNumber(startingBlockNumber, endingBlockNumber uint64) []*protos.Block
	GetBlockchainSize() uint64
	VerifyChain(highBlock, lowBlock uint64) (uint64, error)
}
```

# Engine 具体思路


### <a name="pointintime"></a>时间点查询

在抽象的时间方面，有三种查询对chaincode和应用开发者非常重要：

1. 查询一个key的最新value。(类型: 当前; 例如，现在Alice的账户里有多少钱？)
2. 查询一个key在某一特定时间的value。(类型: 历史; 例如，在上个月Alice的账户余额是多少？)
3. 查询一个key随着时间变化的所有value。(类型: 系列; 例如，生成Alice的交易列表。)

当制定一个查询时，开发人员将受益于过滤、预测以及交易间关联等功能。考虑下面的例子：

1. 简单过滤：查询所有上个月余额低于100美元的账户。
2. 复杂过滤：查询所有Trudy的，发生在叙利亚或伊拉克的，总量大于一个阈值的，其他部分的名称匹配一个正则表达式的交易。
3. 关联：确定是否Alice在同一天同一加油站买过不止一次油。将此信息输入诈骗检测模型。
4. 预测: 查询Alice最后10个交易的市、州、国和金额。这些信息被输入风险/欺诈检测模型。


### <a name="querylanguage"></a>查询语言

开发一种查询语言来支持不同的查询范围并不简单。面临的挑战：

1. 随着开发人员需求的增长扩展查询语言。截至目前，开发人员的请求已经不大。随着Hyperledger项目的用户增加，查询将更复杂。
2. 两个不太相关的类的查询：
    1. 查询符合约束的单一value。适合现有的SQL和NoSQL语法。
    2. 查询满足约束的交易的一个chain或者多个chain。适合图形查询语言，如 Neo4J's Cypher or SPARQL。
