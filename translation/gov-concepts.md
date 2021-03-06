# Concepts

_免责声明：这是正在进行的工作。 机制很容易发生变化。_

治理过程分为以下几个步骤：

- **Proposal submission:** 提案和押金提交给区块链。
- **Vote:** 一旦押金达到一定值（`MinDeposit`），提案就进入投票阶段，绑定Atom的持有者可以发送`TxGovVote`交易投票给提案
- 如果建议涉及软件升级
  - **Signal:** 节点开始发出信号表明他们已经准备好升级到新的版本
  - **Switch:** 一旦超过75%的节点表示已经准备好切换到新的版本，他们的软件就会自动升级到新的版本。

## Proposal submission

### Right to submit a proposal

任何Atom的持有者，无论是否有担保，都可以通过`TxGovProposal`操作提交提案，提案提交后可以通过`proposalID`进行标识。

### Proposal types

在治理模块的初始版本中，有两种类型的提案：

- `PlainTextProposal` 所有不涉及代码修改的都属于这一种，例如：民意调查将使用`PlainTextProposal`类型的提案
- `SoftwareUpgradeProposal`.如果接受，节点将根据提议更新他们的软件，他们必须按照[Software Upgrade](#software-upgrade)的描述分2步来完成升级，可以通过`PlainTextProposal`提案来讨论升级路线，但实际的软件升级必须按照`SoftwareUpgradeProposal`提案进行执行。

其他模块可以根据实际的提案类型和处理程序来扩展治理模块，这些类型通过管理模块（例如：`ParamChangeProposal`）注册和处理，然后在投票通过时执行相应提案的处理程序，此自定义处理程序可执行任意状态改变。

## Deposit

为了防止垃圾提案，必须以 `MinDeposit`参数来定义提案的最小押金，当提案押金小于 `MinDeposit`的时候，投票期不会开始。

当提案提交的时候，必须附加上严格为正数的押金，但押金不必要大于`MinDeposit`，提交者不需要自己支付全部押金，如果提案的押金低于`MinDeposit`，其他代币持有者可以通过发送`Deposit`命令来为提案增加押金，在提案定稿之前（通过或拒绝），这笔押金会存放在模块的第三方托管中。

当提案押金达到`MinDeposit`的时候提案进入投票期，如果在`MaxDepositPeriod`之前提案押金没有达到`MinDeposit`，则提案关闭且没有人可以再给提案增加押金。

### Deposit refund and burn

当提案最终确定以后，根据提案的最终统计，押金将会退还或者销毁。

- 如果提案通过或者被拒绝，则返回押金给各自的提交押金的账户
- 当提案被大多数人强烈反对，则直接销毁

## Vote

### Participants

_参与者_ 是有权对提案进行投票的用户，在Cosmos Hub上，参与者是有投票的用户，没有投票的Atom持有人是无权参加社区治理的，但是他们可以提交提案和给提案增加押金。

请注意，在某些情况下，某些 _参与者_ 可以禁止参与某些提案的投票。

- 当提案进入投票期以后，_参与者_ 投票或者赎回的Atom.
- 提案进入投票期以后，_参与者_ 成为验证人

这不会阻止 _参与者_ 使用投票给其他节点的Atom进行投票，例如，_参与者_ 在提案进入投票期之前对节点A进行投票，而在提案进入投票期以后将其他Atom 投票给节点B，则仅禁止在节点B上投票的Atom进行投票。

### Voting period

提案押金达到`MinDeposit`，立刻进入投票期，我们将投票期定义为开始投票期到结束投票期的时间，`投票期`始终少于`赎回投票期`，以防止出现双重投票，`投票期`初始设置为2周

### Option set

提案的选项集是指参与者在投票时可以选择的选项集。

初始选项集包括以下选项：

- `Yes`
- `No`
- `NoWithVeto`
- `Abstain`

`NoWithVeto`被视为 `No` 包含 `Veto` 投票. `Abstain`选项使选民可以表示自己无意投票赞成和返回但是愿意接受投票结果。

_注意：从用户界面中，对于紧急建议，我们可能应该添加一个"不紧急"选项，以进行" NoWithVeto"投票。_

### Quorum

法定人数定义为为使结果有效而需要对提案施加的投票权的最小百分比。

### Threshold

阈值定义为接受提案的"赞成"票（不包括"弃权"票）的最低比例。

最初，将阈值设置为50％，如果超过1/3票（不包括弃权票）为NoWithVeto票，则可以否决。 这意味着，如果投票期结束时"是"（不包括"弃权"票）的投票比例高于50％，并且如果"否满票"的投票率低于1/3（不包括投"弃权"票），则接受提案。

如果提案符合特殊条件，可以在投票期结束前通过，即，如果`Yes`占`InitTotalVotingPower`的2/3，即使投票期未结束，提案也会通过，`InitTotalVotingPower`是提案投票期开始时所有投票Atom的总投票权，存在这种情况是为了让网络在紧急情况下可以快速做出反应。

### Inheritance

如果投票人不投票，节点将继承其投票权

- 如果投票者在节点投票前进行投票，则节点不会继承其投票权
- 如果投票者在节点之后进行投票，则它会覆盖节点的投票，如果投票比较紧急，则可能在投票人有机会覆盖节点投票之前结束投票期，这不是问题，因为提案需要在投票期结束前通过总投票权2/3的投票，如果合起来有2/3的节点，他们仍然可以审查代表的票数。

### Validator's punishment for non-voting

目前为止，节点没有因为不投票而受到惩罚

### Governance address

稍后，我们可能会添加对某些模块的tx签名的许可密钥，对于MVP， `Governance address`将会是创建账户时生成的主要验证者地址。该地址与负责签署共识消息的Tendermint PrivKey 对应不同的PrivKey，因此，节点不必使用敏感的Tendermint PrivKey进行签署交易

## Software Upgrade

如果提案的类型为" SoftwareUpgradeProposal"，则节点需要将其软件升级到投票的新版本。 此过程分为两个步骤。

### Signal

在一个`SoftwareUpgradeProposal`提案通过以后，节点应该下载并安装新的升级程序，并运行老的程序。节点下载并安装升级程序之后，他将通过在提案的 _precommits_ 中包含`proposalID` 来开始向网络发出进行切换的信号（注意：确认是否需要在以前的precommit中？）

注意：每一个_precommit_仅有一个信号槽，如果在短时间内接收到几个`SoftwareUpgradeProposals`提案，则会形成一个管道，并将按照接收顺序依次执行。

### Switch

一旦一个块包含了超过2/3的" precommits_"，并发出了一个通用的" SoftwareUpgradeProposal"信号，则所有节点（包括验证器节点，未验证的完整节点和轻节点）都将切换到该软件的新版本。

_注意：不清楚如何以编程方式处理翻转_