# State

## Parameters and base types

"参数"定义了运行选票的规则。 在任何给定时间只能设置一个活动参数。 如果治理要更改参数集（以修改值或添加/删除参数字段），则必须创建一个新的参数集，而先前的参数集将变为非活动状态。

```go
type DepositParams struct {
  MinDeposit        sdk.Coins  //  Minimum deposit for a proposal to enter voting period.
  MaxDepositPeriod  time.Time  //  Maximum period for Atom holders to deposit on a proposal. Initial value: 2 months
}
```

```go
type VotingParams struct {
  VotingPeriod      time.Time  //  Length of the voting period. Initial value: 2 weeks
}
```

```go
type TallyParams struct {
  Quorum            sdk.Dec  //  Minimum percentage of stake that needs to vote for a proposal to be considered valid
  Threshold         sdk.Dec  //  Minimum proportion of Yes votes for proposal to pass. Initial value: 0.5
  Veto              sdk.Dec  //  Minimum proportion of Veto votes to Total votes ratio for proposal to be vetoed. Initial value: 1/3
}
```

参数存储在全局`GlobalParams` KVStore中。

此外，我们介绍了一些基本类型：

```go
type Vote byte

const (
    VoteYes         = 0x1
    VoteNo          = 0x2
    VoteNoWithVeto  = 0x3
    VoteAbstain     = 0x4
)

type ProposalType  string

const (
    ProposalTypePlainText       = "Text"
    ProposalTypeSoftwareUpgrade = "SoftwareUpgrade"
)

type ProposalStatus byte


const (
    StatusNil           ProposalStatus = 0x00
    StatusDepositPeriod ProposalStatus = 0x01  // Proposal is submitted. Participants can deposit on it but not vote
    StatusVotingPeriod  ProposalStatus = 0x02  // MinDeposit is reached, participants can vote
    StatusPassed        ProposalStatus = 0x03  // Proposal passed and successfully executed
    StatusRejected      ProposalStatus = 0x04  // Proposal has been rejected
    StatusFailed        ProposalStatus = 0x05  // Proposal passed but failed execution
)
```

## Deposit

```go
  type Deposit struct {
    Amount      sdk.Coins       //  Amount of coins deposited by depositor
    Depositor   crypto.address  //  Address of depositor
  }
```

## ValidatorGovInfo

计算时在临时映射中使用此类型

```go
  type ValidatorGovInfo struct {
    Minus     sdk.Dec
    Vote      Vote
  }
```

## Proposals

`Proposal`对象用来记票并跟踪投票提案状态，他们包含`Content`标识本提案的内容以及其他字段，它们是治理流程的可变阶段

```go
type Proposal struct {
    Content  // Proposal content interface

    ProposalID       uint64
    Status           ProposalStatus  // Status of the Proposal {Pending, Active, Passed, Rejected}
    FinalTallyResult TallyResult     // Result of Tallies

    SubmitTime     time.Time  // Time of the block where TxGovSubmitProposal was included
    DepositEndTime time.Time  // Time that the Proposal would expire if deposit amount isn't met
    TotalDeposit   sdk.Coins  // Current deposit on this proposal. Initial value is set at InitialDeposit

    VotingStartTime time.Time  //  Time of the block where MinDeposit was reached. -1 if MinDeposit is not reached
    VotingEndTime   time.Time  // Time that the VotingPeriod for this proposal will end and votes will be tallied
}
```

```go
type Content interface {
    GetTitle() string
    GetDescription() string
    ProposalRoute() string
    ProposalType() string
    ValidateBasic() sdk.Error
    String() string
}
```

提案中的`Content`是一个接口，它包含提案的相关信息例如标题，描述和任何明显的改变同样这种`Content`类型可以由任何模块实现。`Content`的`ProposalRoute`返回一个字符串，该字符串必须用在治理模块路由 `Content`的`Handler`，这使治理模块能够实现任何模块实现的提议逻辑，如果提案通过，则执行处理程序，仅当处理程序成功时，状态才会持久，并且提案会最终通过，否则，提案会被拒绝。

```go
type Handler func(ctx sdk.Context, content Content) sdk.Error
```

`Handler` 负责处理提案并根据提案内容修改指定状态，在提案通过时`EndBlock`期间进行执行。

我们还提到了一种更新给定提案的理货的方法：

```go
  func (proposal Proposal) updateTally(vote byte, amount sdk.Dec)
```

## Stores

_Stores是多商店中的KVStores。 查找商店的关键是list中的第一个参数_

我们将使用一个KVStore`Governance`存储两个映射：

-从` proposalID |'proposal'`到`Proposal`的映射。
-从` proposalID |'addresses'| address`到`Vote`的映射。 通过这种映射，我们可以通过在proposalID：addresses上进行范围查询来查询对提案投票的所有地址及其投票。

出于伪代码的目的，这是我们将在商店中读取或写入的两个函数：

- `load(StoreKey, Key)`: 在键为`StoreKey`的储存中查找键为`Key`的项目
- `store(StoreKey, Key, value)`: 在键为`StoreKey`的储存中储存键为`Key`的值

## Proposal Processing Queue

**Store:**

- `ProposalProcessingQueue`: 一个队列 `queue[proposalID]` 包含所有押金达到`MinDeposit`提案的 `ProposalIDs`  . 在每个 `EndBlock`,投票期结束的所有提案都将得到处理，为了处理提案，应用程序会计算票数，然后检查出块节点中每一个节点是否已经投票. 如果提案通过，则押金将退还。 最后，执行提案内容"处理程序"。

还有ProposalProcessingQueue的伪代码：

```go
  in EndBlock do

    for finishedProposalID in GetAllFinishedProposalIDs(block.Time)
      proposal = load(Governance, <proposalID|'proposal'>) // proposal is a const key

      validators = Keeper.getAllValidators()
      tmpValMap := map(sdk.AccAddress)ValidatorGovInfo

      // Initiate mapping at 0. This is the amount of shares of the validator's vote that will be overridden by their delegator's votes
      for each validator in validators
        tmpValMap(validator.OperatorAddr).Minus = 0

      // Tally
      voterIterator = rangeQuery(Governance, <proposalID|'addresses'>) //return all the addresses that voted on the proposal
      for each (voterAddress, vote) in voterIterator
        delegations = stakingKeeper.getDelegations(voterAddress) // get all delegations for current voter

        for each delegation in delegations
          // make sure delegation.Shares does NOT include shares being unbonded
          tmpValMap(delegation.ValidatorAddr).Minus += delegation.Shares
          proposal.updateTally(vote, delegation.Shares)

        _, isVal = stakingKeeper.getValidator(voterAddress)
        if (isVal)
          tmpValMap(voterAddress).Vote = vote

      tallyingParam = load(GlobalParams, 'TallyingParam')

      // Update tally if validator voted they voted
      for each validator in validators
        if tmpValMap(validator).HasVoted
          proposal.updateTally(tmpValMap(validator).Vote, (validator.TotalShares - tmpValMap(validator).Minus))



      // Check if proposal is accepted or rejected
      totalNonAbstain := proposal.YesVotes + proposal.NoVotes + proposal.NoWithVetoVotes
      if (proposal.Votes.YesVotes/totalNonAbstain > tallyingParam.Threshold AND proposal.Votes.NoWithVetoVotes/totalNonAbstain  < tallyingParam.Veto)
        //  proposal was accepted at the end of the voting period
        //  refund deposits (non-voters already punished)
        for each (amount, depositor) in proposal.Deposits
          depositor.AtomBalance += amount

        stateWriter, err := proposal.Handler()
        if err != nil
            // proposal passed but failed during state execution
            proposal.CurrentStatus = ProposalStatusFailed
         else
            // proposal pass and state is persisted
            proposal.CurrentStatus = ProposalStatusAccepted
            stateWriter.save()
      else
        // proposal was rejected
        proposal.CurrentStatus = ProposalStatusRejected

      store(Governance, <proposalID|'proposal'>, proposal)
```