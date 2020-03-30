# Messages

在本节中，我们描述对抵押消息的处理以及对状态的相应更新。 每个消息指定的所有创建/修改的状态对象都在[state](./02_state_transitions.md)部分中定义。

## MsgCreateValidator

创建节点使用 `MsgCreateValidator`消息

```go
type MsgCreateValidator struct {
    Description    Description
    Commission     Commission

    DelegatorAddr  sdk.AccAddress
    ValidatorAddr  sdk.ValAddress
    PubKey         crypto.PubKey
    Delegation     sdk.Coin
}
```

如果出现以下情况，此消息将失败：

- 已经使用该操作员地址注册了另一个节点
- 已经使用该操作员公钥注册了另一个节点
- 抵押代币不是系统主代币
- 佣金参数不符合规定:
  - `MaxRate` 大于1或小于0
  - 初始 `Rate` 是负数或者大于 `MaxRate`
  - 初始 `MaxChangeRate` 是负数或者大于 `MaxRate`
- 描述字段太长

这个消息在合适的索引处创建并存储`Validator`对象,使用系统主代币进行初始投票,节点最初的状态是unbonded 但可以在第一个块结束时变为 bonded

## MsgEditValidator

节点的`Description`, `CommissionRate`可以通过`MsgEditCandidacy`消息进行修改

```go
type MsgEditCandidacy struct {
    Description     Description
    ValidatorAddr   sdk.ValAddress
    CommissionRate  sdk.Dec
}
```

如果出现以下情况，此消息将失败：

- `CommissionRate` 小于0或者大于 `MaxRate`
- `CommissionRate` 已经在24小时内更新过了
- `CommissionRate` 大于 `MaxChangeRate`
- 描述字段太长

这个消息将会存储更新过后的`Validator`对象

## MsgDelegate

在此消息中，投票人提供代币，作为回报，接收节点给投票人的一定股份称为`Delegation.Shares`

```go
type MsgDelegate struct {
  DelegatorAddr sdk.AccAddress
  ValidatorAddr sdk.ValAddress
  Amount        sdk.Coin
}
```

如果出现以下情况，此消息将失败：

- 节点不存在
- 节点被关小黑屋
- 代币、面额和参数`params.BondDenom`不匹配

如果投票信息不存在，则将该信息对象创建做为该消息处理的一部分，否则，将更新现有的投票信息。

## MsgBeginUnbonding

取消投票消息允许投票人开始赎回自己的投票。

```go
type MsgBeginUnbonding struct {
  DelegatorAddr sdk.AccAddress
  ValidatorAddr sdk.ValAddress
  Amount         sdk.Coin
}
```

如果出现以下情况，此消息将失败：

- 投票信息不存在
- 节点不存在
- 投票的份额少于要赎回的代币数额
- 现有的`UnbondingDelegation`已经超过`params.MaxEntries`定义的最大值了
- 代币面额和`params.BondDenom`定义的不同

处理此消息后，将发生以下操作：

- 节点的总投票股份和投票人的股份都会根据消息的`SharesAmount`而减少
- 计算股份的代币价值，减少节点上投票代币的数量
- 当这些代币减少的时候,根据节点状态的不同:
  - `Bonded` - 将他们更新到 `UnbondingDelegation`条目中,并增加一个冻结期,减少公共投票池中的股份以及更新未投票代币的数量
  - `Unbonding` -将他们更新到 `UnbondingDelegation`条目中,并增加一个最小冻结期`UnbondingMinTime`.
  - `Unbonded` -将代币打给`DelegatorAddr`
- 如果投票中没有多余的代币和股份,则删除这条记录
  - 在这种情况下,如果是节点本身的操作,则将节点关小黑屋

## MsgBeginRedelegate

转投命令允许投票人切换投票节点,一旦发起转投消息,转投将在EndBlock的时候生效

```go
type MsgBeginRedelegate struct {
  DelegatorAddr    sdk.AccAddress
  ValidatorSrcAddr sdk.ValAddress
  ValidatorDstAddr sdk.ValAddress
  Amount           sdk.Coin
}
```

如果出现以下情况，此消息将失败：

- 投票信息不存在
- 源节点或者目标节点不存在
- 转投的份额小于1个代币的份额
- 源节点收到还未生效的转投 (也就是说转投是不能传递的)
- 转投的份额超过了`params.MaxEntries`定义的最大值
- 代币的面额和`params.BondDenom`定义的面额不同

处理此消息后，将发生以下操作：

- 源节点的股份和投票人在源节点上的股份都会减少
- 计算股份的价值,去掉源节点上面抵押股份的代币
- 根据源节点的状态:
  - `Bonded` - 在转投里面增加一个条目,完成时间是一个完整的冻结期,更新投票池中的份额并增加解绑池中的份额
  - `Unbonding` - 在转投池中增加一个条目,完成时间是最小解绑时间`UnbondingMinTime`
  - `Unbonded` - 不做任何操作
- 将代币投给目标节点 ,可能会将票移回绑定状态
- 如果源节点的投票目录里面没有份额,则将此目录删除
  - 在这种情况下如果是验证者自己的操作,则将节点关进小黑屋.