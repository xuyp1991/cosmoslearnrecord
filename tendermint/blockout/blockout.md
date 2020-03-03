# 本文档是学习cosmos出块的记录

## 相关函数的调用顺序
![Image text]([https://github.com/xuyp1991/cosmoslearnrecord/blob/master/tendermint/blockout/tendmint_blockout.png])

## 相关函数的说明

### 入口

根据总结的图片,入口有3个    OnStart,startRoutines和handleMsg,关于入口这里只有一些猜想

+  OnStart实现cmn.Service。它通过WAL加载最新状态，并启动超时和接收例程。
+  timeoutRoutine：在tickChan上接收超时请求，在tockChan上触发超时   receiveRoutine：序列化提案，块部分，投票的处理； 协调状态转换
+  handleMsg  处理service上面的msg信息


### 主协程

OnStart和startRoutines 都会调用receiveRoutine函数,而且是另外启动协程来运行该函数

OnStart

```state.go
go cs.receiveRoutine(0)
```

startRoutines

```state.go
go cs.receiveRoutine(maxSteps)
```

receiveRoutine函数的作用   

+ receiveRoutine处理可能导致状态转换的消息。
+ 它的参数（n）是退出前要处理的消息数-永远使用0
+ 它保留RoundState，并且是唯一对其进行更新的东西。
+ 更新（状态转换）发生在超时，完整建议和2/3多数情况下。
+ 必须先锁定ConsensusState，然后才能更新任何内部状态。

receiveRoutine中调用的函数是:handleTimeout和handleTxsAvailable,关于这两个函数的猜想:

+ handleTimeout处理区块到时间的相关内容,也就是到一定时间必须要出块应该会调用这个函数
+ handleTxsAvailable处理相关事务的信息,当需要出块的时候调用这个并验证所有事务的可用性

handleTimeout和handleTxsAvailable均会调用函数enterNewRound

### enterNewRound

状态功能:

由handleTimeout和handleMsg在内部使用以进行状态转换

+ 输入：`timeoutNewHeight`，以startTime（commitTime + timeoutCommit）表示，或者，如果SkipTimeout == true，则从（height，round-1）接收所有预提交
+ 输入：`timeoutPrecommits`在（height，round-1）进行任何+2/3 precommits后，
+ 输入：+2/3 precommits 在（height，round-1）预提交nil
+ 输入：+2/3 prevotes给任何人，或者+2/3 precommits 来自任意（高度，回合）的块

注意：cs.StartTime已设置为高度。

上面的描述有点难懂,这个函数最终会调用enterPropose函数.

### enterPropose

+ 输入（CreateEmptyBlocks）：从enterNewRound（height，round）
+ 输入（CreateEmptyBlocks，CreateEmptyBlocksInterval> 0）：在EnterNewRound（height，round）之后，在CreateEmptyBlocksInterval超时后
+ 输入（！CreateEmptyBlocks）：在enterNewRound（height，round）之后，一旦txs进入内存池
  
enterPropose 最终调用defaultDecideProposal

### defaultDecideProposal

defaultDecideProposal生成区块以及区块的摘要,并根据区块和摘要生成提案并广播出去    cosmos将未被确认的区块称之为提案

关键代码

```state.go
 block, blockParts = cs.createProposalBlock()
  propBlockId := types.BlockID{Hash: block.Hash(), PartsHeader: blockParts.Header()}
proposal := types.NewProposal(height, round, cs.ValidRound, propBlockId)//round这个参数是否有意义?
    cs.sendInternalMessage(msgInfo{&ProposalMessage{proposal}, ""})
    for i := 0; i < blockParts.Total(); i++ {
        part := blockParts.GetPart(i)
        cs.sendInternalMessage(msgInfo{&BlockPartMessage{cs.Height, cs.Round, part}, ""})
    }
```

+ createProposalBlock 函数生成区块以及区块摘要
+ NewProposal 生成提案
+ sendInternalMessage 将提案或者摘要信息广播出去

### createProposalBlock

创建下一个要提议的块并返回。 我们确实只需要退回零件，但是为方便起见，返回了该方框，因此我们可以记录提案方框。 发生错误时返回nil块。 注意：为了清楚起见，请使其无副作用。

最终调用  CreateProposalBlock

### CreateProposalBlock

CreateProposalBlock调用state.MakeBlock，其中包含来自evpool的证据和来自mempool的txs。 最大字节数必须足够大以适合提交。 为了最大尺寸的证据，最多对块空间的1/10进行了涂层。 其余的给txs，直到gas。

### MakeBlock

MakeBlock使用给定的tx，提交和证据从当前状态构建一个块．

