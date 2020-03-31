# Messages

## Proposal Submission

任何Atom持有人都可以通过`TxGovSubmitProposal`事务提交提案。

```go
type TxGovSubmitProposal struct {
    Content        Content
    InitialDeposit sdk.Coins
    Proposer       sdk.AccAddress
}
```

`TxGovSubmitProposal`消息的` Content`必须在管理模块中设置适当的路由器。

**State modifications:**

- 生成新的 `proposalID`
- 创建新的 `Proposal`
- 初始化 `Proposals` 属性
- 通过 `InitialDeposit`减少发提案的人的余额
- 如果达到 `MinDeposit`:
  - 将 `proposalID` 添加到 `ProposalProcessingQueue`
- 从 `Proposer`中转账`InitialDeposit` 到 gov模块的 `ModuleAccount`

`TxGovSubmitProposal`事务可以根据以下伪代码进行处理

```go
// PSEUDOCODE //
// Check if TxGovSubmitProposal is valid. If it is, create proposal //

upon receiving txGovSubmitProposal from sender do

  if !correctlyFormatted(txGovSubmitProposal)
    // check if proposal is correctly formatted. Includes fee payment.
    throw

  initialDeposit = txGovSubmitProposal.InitialDeposit
  if (initialDeposit.Atoms <= 0) OR (sender.AtomBalance < initialDeposit.Atoms)
    // InitialDeposit is negative or null OR sender has insufficient funds
    throw

  if (txGovSubmitProposal.Type != ProposalTypePlainText) OR (txGovSubmitProposal.Type != ProposalTypeSoftwareUpgrade)

  sender.AtomBalance -= initialDeposit.Atoms

  depositParam = load(GlobalParams, 'DepositParam')

  proposalID = generate new proposalID
  proposal = NewProposal()

  proposal.Title = txGovSubmitProposal.Title
  proposal.Description = txGovSubmitProposal.Description
  proposal.Type = txGovSubmitProposal.Type
  proposal.TotalDeposit = initialDeposit
  proposal.SubmitTime = <CurrentTime>
  proposal.DepositEndTime = <CurrentTime>.Add(depositParam.MaxDepositPeriod)
  proposal.Deposits.append({initialDeposit, sender})
  proposal.Submitter = sender
  proposal.YesVotes = 0
  proposal.NoVotes = 0
  proposal.NoWithVetoVotes = 0
  proposal.AbstainVotes = 0
  proposal.CurrentStatus = ProposalStatusOpen

  store(Proposals, <proposalID|'proposal'>, proposal) // Store proposal in Proposals mapping
  return proposalID
```

## Deposit

提案提交以后，如果`Proposal.TotalDeposit < ActiveParam.MinDeposit`，ATOM的持有者可以通过发送`TxGovDeposit`事务的方式来给提案添加押金

```go
type TxGovDeposit struct {
  ProposalID    int64       // ID of the proposal
  Deposit       sdk.Coins   // Number of Atoms to add to the proposal's deposit
}
```

**State modifications:**

- 减少发送 `deposit`事务的人的余额
- 在`proposal.Deposits`中添加发送者的的`deposit`
- 通过发送者的 `deposit`增加 `proposal.TotalDeposit` 
- 如果达到 `MinDeposit`
  - 将 `proposalID` 添加到 `ProposalProcessingQueueEnd`
- 将 `proposer`的`Deposit` 转到`ModuleAccount`

TxGovDeposit事务必须经过许多检查才能有效。这些检查在以下伪代码中概述。

```go
// PSEUDOCODE //
// Check if TxGovDeposit is valid. If it is, increase deposit and check if MinDeposit is reached

upon receiving txGovDeposit from sender do
  // check if proposal is correctly formatted. Includes fee payment.

  if !correctlyFormatted(txGovDeposit)
    throw

  proposal = load(Proposals, <txGovDeposit.ProposalID|'proposal'>) // proposal is a const key, proposalID is variable

  if (proposal == nil)
    // There is no proposal for this proposalID
    throw

  if (txGovDeposit.Deposit.Atoms <= 0) OR (sender.AtomBalance < txGovDeposit.Deposit.Atoms) OR (proposal.CurrentStatus != ProposalStatusOpen)

    // deposit is negative or null
    // OR sender has insufficient funds
    // OR proposal is not open for deposit anymore

    throw

  depositParam = load(GlobalParams, 'DepositParam')

  if (CurrentBlock >= proposal.SubmitBlock + depositParam.MaxDepositPeriod)
    proposal.CurrentStatus = ProposalStatusClosed

  else
    // sender can deposit
    sender.AtomBalance -= txGovDeposit.Deposit.Atoms

    proposal.Deposits.append({txGovVote.Deposit, sender})
    proposal.TotalDeposit.Plus(txGovDeposit.Deposit)

    if (proposal.TotalDeposit >= depositParam.MinDeposit)
      // MinDeposit is reached, vote opens

      proposal.VotingStartBlock = CurrentBlock
      proposal.CurrentStatus = ProposalStatusActive
      ProposalProcessingQueue.push(txGovDeposit.ProposalID)

  store(Proposals, <txGovVote.ProposalID|'proposal'>, proposal)
```

## Vote

一旦达到`ActiveParam.MinDeposit`，便开始投票期，从这时开始，Atom的投票人可以通过发送 `TxGovVote`来进行他们对这个提案的投票。

```go
  type TxGovVote struct {
    ProposalID           int64         //  proposalID of the proposal
    Vote                 byte          //  option from OptionSet chosen by the voter
  }
```

**State modifications:**

- 记录投票的发起者

  _注意：此消息的gas费用必须考虑到EndBlocker中未来的投票数_

接下来是处理`TxGovVote`事务的伪代码：

```go
  // PSEUDOCODE //
  // Check if TxGovVote is valid. If it is, count vote//

  upon receiving txGovVote from sender do
    // check if proposal is correctly formatted. Includes fee payment.

    if !correctlyFormatted(txGovDeposit)
      throw

    proposal = load(Proposals, <txGovDeposit.ProposalID|'proposal'>)

    if (proposal == nil)
      // There is no proposal for this proposalID
      throw


    if  (proposal.CurrentStatus == ProposalStatusActive)


        // Sender can vote if
        // Proposal is active
        // Sender has some bonds

        store(Governance, <txGovVote.ProposalID|'addresses'|sender>, txGovVote.Vote)   // Voters can vote multiple times. Re-voting overrides previous vote. This is ok because tallying is done once at the end.
```