# State

## LastTotalPower

LastTotalPower跟踪在上一个结束块期间记录的绑定代币的总量。

- LastTotalPower: `0x12 -> amino(sdk.Int)`

## Params

参数是模块范围的配置结构，该结构存储系统参数并定义放样模块的整体功能。

- Params: `Paramsspace("staking") -> amino(params)`

```go
type Params struct {
    UnbondingTime time.Duration // time duration of unbonding
    MaxValidators uint16        // maximum number of validators
    MaxEntries    uint16        // max entries for either unbonding delegation or redelegation (per pair/trio)
    BondDenom     string        // bondable coin denomination
}
```

## Validator

验证者可以具有以下三种状态之一

- `Unbonded`:节点不在出块节点集中。 他们无法签署区块，也无法获得奖励。 他们可以接受投票。
- `Bonded`:当节点获得足够的投票,他们会在[`EndBlock`](./05_end_block.md#validator-set-changes) 的时候加入出块节点集,节点的状态会更新为`Bonded`.它们可以签名区块并获得奖励,他们也可以获得更多的投票,他们也会因为干坏事而收到惩罚,投票人取消投票必须等待一个UnbondingTime的时间,UnbondingTime是链上的一个参数.在此期间如果出块节点有犯罪行为,投票人依然会受到惩罚.
- `Unbonding`:当一个节点因为自己选择或者被惩罚或者被注销等原因离开出块节点集,它就进入`Unbonding`状态,所有投它票的人都会开始等待UnbondingTime后才能从`BondedPool`中取回他们的代币.节点对象主要由`OperatorAddr`对象进行储存和访问,`OperatorAddr`是SDK上面节点的地址,每个节点都有2个索引以便节点更新或者节点惩罚的时候能快速定位到节点,节点还维护了一个特殊索引`LastValidatorPower`,这个索引在每个块中都会保持不变,和其他索引不同它记录了区块的出块节点.
  
- Validators: `0x21 | OperatorAddr -> amino(validator)`
- ValidatorsByConsAddr: `0x22 | ConsAddr -> OperatorAddr`
- ValidatorsByPower: `0x23 | BigEndian(ConsensusPower) | OperatorAddr -> OperatorAddr`
- LastValidatorsPower: `0x11 OperatorAddr -> amino(ConsensusPower)`

`Validators` 是主索引,它保证了每一个操作者都有且仅有一个关联的节点,节点的publickey在未来是可以修改的.投票人可以把票投给操作者,而不需要关心节点公钥的改变

`ValidatorByConsAddr`是一个附加索引,可以用于惩罚机制的查找,当Tenderminter提交证据的时候,它提供节点地址,所以这个索引需要提供操作员,注意`ConsAddr`是可以从节点的`ConsPubKey`派生出来.

`ValidatorsByPower`是一个附加的索引,它提供了一个根据投票数排序的节点的列表,使我们可以根据这个列表快速定位哪些是出块节点.这里ConsensusPower是给节点投票的币/10^6.注意,状态是`Jailed`的节点是不会保存在这个索引里面的.

`LastValidatorsPower`是一个特殊的索引,提供最后一个区块的节点的总投票数的历史列表,这个索引在出块期间保持不变,但是在 [`EndBlock`](./05_end_block.md)会更新.

所有节点的状态信息都保存在`Validator`结构体里面

```go
type Validator struct {
    OperatorAddress         sdk.ValAddress  // address of the validator's operator; bech encoded in JSON
    ConsPubKey              crypto.PubKey   // the consensus public key of the validator; bech encoded in JSON
    Jailed                  bool            // has the validator been jailed from bonded status?
    Status                  sdk.BondStatus  // validator status (bonded/unbonding/unbonded)
    Tokens                  sdk.Int         // delegated tokens (incl. self-delegation)
    DelegatorShares         sdk.Dec         // total shares issued to a validator's delegators
    Description             Description     // description terms for the validator
    UnbondingHeight         int64           // if unbonding, height at which this validator has begun unbonding
    UnbondingCompletionTime time.Time       // if unbonding, min time for the validator to complete unbonding
    Commission              Commission      // commission parameters
    MinSelfDelegation       sdk.Int         // validator's self declared minimum self delegation
}

type Commission struct {
    CommissionRates
    UpdateTime time.Time // the last time the commission rate was changed
}

CommissionRates struct {
    Rate          sdk.Dec // the commission rate charged to delegators, as a fraction
    MaxRate       sdk.Dec // maximum commission rate which validator can ever charge, as a fraction
    MaxChangeRate sdk.Dec // maximum daily increase of the validator commission, as a fraction
}

type Description struct {
    Moniker          string // name
    Identity         string // optional identity signature (ex. UPort or Keybase)
    Website          string // optional website link
    SecurityContact  string // optional email for security contact
    Details          string // optional details
}
```

## Delegation

通过将" DelegatorAddr"（投票人的地址）与" ValidatorAddr"结合使用来标识投票信息。在内存中对投票信息的索引如下：

- Delegation: `0x31 | DelegatorAddr | ValidatorAddr -> amino(delegation)`

用户可以将代币投票给节点,这种情况下他们的资金将存放在`Delegation`数据结构中,它由每一个投票人所持有,并和一个节点的股份相关联,交易的发送者是持币人.

```go
type Delegation struct {
    DelegatorAddr sdk.AccAddress
    ValidatorAddr sdk.ValAddress
    Shares        sdk.Dec        // delegation shares received
}
```

### Delegator Shares

当投票人把一个代币投票给节点的时候,节点会根据动态汇率给他们分配一定数额的股份,该份额根据节点上投票的代币总数和已经发行的股份总数计算而来.

`Shares per Token = validator.TotalShares() / validator.Tokens()`

只有份额会记录在DelegationEntry中,当投票人取消投票的时候,将更加他们持有的股份份额和你汇率来计算他们受到的代币数量.

`Tokens per Share = validator.Tokens() / validatorShares()`

这些份额是一种会计机制,它们是不可代替资产, 这种机制简化了围绕惩罚机制的记账,它可以通过削减节点的总投票币数.而不是削减每一个投票人的投票数,从而有效减少每一个投票人的投票价值.

## UnbondingDelegation

投票是可以取消的,但必须有一个`UnbondingDelegation`时间,在此期间如果检测到节点有违规行为,投票的份额依然会被削减.

`UnbondingDelegation`在内存中的结构是:

- UnbondingDelegation: `0x32 | DelegatorAddr | ValidatorAddr ->
   amino(unbondingDelegation)`
- UnbondingDelegationsFromValidator: `0x33 | ValidatorAddr | DelegatorAddr ->
   nil`

第一个索引用于查询,用于查询一个投票人所有待取消的投票,第二个索引用于惩罚机制,用于查询一个即将被惩罚的节点所有待取消的投票

每次取消投票的时候,都会创建一个UnbondingDelegation对象

```go
type UnbondingDelegation struct {
    DelegatorAddr sdk.AccAddress             // delegator
    ValidatorAddr sdk.ValAddress             // validator unbonding from operator addr
    Entries       []UnbondingDelegationEntry // unbonding delegation entries
}

type UnbondingDelegationEntry struct {
    CreationHeight int64     // height which the unbonding took place
    CompletionTime time.Time // unix time for unbonding completion
    InitialBalance sdk.Coin  // atoms initially scheduled to receive at completion
    Balance        sdk.Coin  // atoms to receive at completion
}
```

## Redelegation

投票人的投票可以立即转给另外一个节点,但是在这种情况下,在`Redelegation`的块中进行结算,如果该块之前投票的节点被惩罚,投票人的份额依然会被削减.

 `Redelegation`在内存中的索引如下:

 - Redelegations: `0x34 | DelegatorAddr | ValidatorSrcAddr | ValidatorDstAddr -> amino(redelegation)`
- RedelegationsBySrc: `0x35 | ValidatorSrcAddr | ValidatorDstAddr | DelegatorAddr -> nil`
- RedelegationsByDst: `0x36 | ValidatorDstAddr | ValidatorSrcAddr | DelegatorAddr -> nil`

第一个索引用于查询,以查询一个投票人所有的转投.第二个索引用于惩罚,根据`ValidatorSrcAddr`地址进行惩罚,第三个索引是基于`ValidatorDstAddr`的投票信息.

每次转投都会重新创建一个投票对象,为了防止"redelegation hopping",以下情况转投不会生效:

- 转投人已经进行了一笔转投操作,其目标是另外一个节点X 并且转投人正在尝试一笔转投,转投的源节点是节点X(简而言之同一个区块不支持对同一笔投票的连续转投)

```go
type Redelegation struct {
    DelegatorAddr    sdk.AccAddress      // delegator
    ValidatorSrcAddr sdk.ValAddress      // validator redelegation source operator addr
    ValidatorDstAddr sdk.ValAddress      // validator redelegation destination operator addr
    Entries          []RedelegationEntry // redelegation entries
}

type RedelegationEntry struct {
    CreationHeight int64     // height which the redelegation took place
    CompletionTime time.Time // unix time for redelegation completion
    InitialBalance sdk.Coin  // initial balance when redelegation started
    Balance        sdk.Coin  // current balance (current value held in destination validator)
    SharesDst      sdk.Dec   // amount of destination-validator shares created by redelegation
}
```

## Queues

所有队列对象均按时间戳排序。 首先将任何队列中使用的时间四舍五入到最接近的纳秒，然后进行排序.所使用的可排序的时间格式是RFC3339Nano的轻微变形，并使用格式字符串`"2006-01-02T15：04：05.000000000"`。注意一下格式:

- 右边补0
- 没有时区信息(使用UTC)
  
  这种情况下,时间戳代表队列元素的生成时间.

  ### UnbondingDelegationQueue

  为了追踪取消投票的进度,保留了取消投票的队列.

  - UnbondingDelegation: `0x41 | format(time) -> []DVPair`

```go
type DVPair struct {
    DelegatorAddr sdk.AccAddress
    ValidatorAddr sdk.ValAddress
}
```

### RedelegationQueue

为了追踪转投的进度,保留了转投的队列.

- UnbondingDelegation: `0x42 | format(time) -> []DVVTriplet`

```go
type DVVTriplet struct {
    DelegatorAddr    sdk.AccAddress
    ValidatorSrcAddr sdk.ValAddress
    ValidatorDstAddr sdk.ValAddress
}
```

### ValidatorQueue

为了追踪unbonding节点的处理进度,保留了ValidatorQueue

- ValidatorQueueTime: `0x43 | format(time) -> []sdk.ValAddress`

作为每个键存储的对象是节点操作员地址的一个数组,从中可以访问到节点对象.通常,希望一个节点和一个给定的时间戳相关联,但是同一个时间戳可能会有多个节点.

## HistoricalInfo

HistoricalInfo对象将在每个块中保存并精简,以便投票持有者保存由投票模块参数`HistoricalEntries`定义的最近`n`个最新的历史信息.

```go
type HistoricalInfo struct {
    Header abci.Header
    ValSet []types.Validator
}
```

在每个BeginBlock函数,staking keeper将提交最近的header,节点也会提交最近的区块在`HistoricalInfo`对象中.节点将会按顺序进行排序以确保他们的顺序性,删除老的HistoricalEntries,仅保留参数定义的`HistoricalInfo`个数