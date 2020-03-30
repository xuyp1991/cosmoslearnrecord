# State Transitions

本文档介绍与以下内容有关的状态转换操作：

1. [Validators](./stake-state-transition.md#validators)
2. [Delegations](./stake-state-transition.md#delegations)
3. [Slashing](./stake-state-transition.md#slashing)

## Validators

节点的状态转换在每一个[`EndBlock`](./05_end_block.md#validator-set-changes) 中进行以便检查出块节点集中的修改

### Unbonded to Bonded

当节点的排名在`ValidatorPowerIndex`中超过最后一个出块节点的时候该状态转换会发生

- 设置 `validator.Status` 为 `Bonded`
- 将代币 `validator.Tokens` 从 `NotBondedTokens` 发送到 `BondedPool` `ModuleAccount`
- 从 `ValidatorByPowerIndex`删除已经存在的记录
- 增加一条新的记录到`ValidatorByPowerIndex`
- 将这个节点的相关更改更细到`Validator`对象中
- 如果存在,删除该节点的 `ValidatorQueue` 中的记录

### Bonded to Unbonding

当节点开始取消绑定过程时，将发生以下操作：

- 将代币 `validator.Tokens` 从 `BondedPool` 转到 `NotBondedTokens` `ModuleAccount`
- 设置 `validator.Status` 为 `Unbonding`
- 删除 `ValidatorByPowerIndex`中已存在的记录
- 增加一条新的记录到`ValidatorByPowerIndex`
- 将这个节点的相关更改更细到`Validator`对象中
- 为该节点增加一条记录到`ValidatorQueue` 

### Unbonding to Unbonded

当`ValidatorQueue` 中的对象从bonded转换成unbonded,节点的状态从Unbonding转换成Unbonded

- 更新该节点的`Validator` 对象
- 设置 `validator.Status` 为 `Unbonded`

### Jail/Unjail

当节点被关小黑屋时，它会从Tendermint集中有效地删除。 此过程也可以颠倒。 发生以下操作：

- 设置 `Validator.Jailed`为True并更新对象
- 如果是管小黑屋则将节点从 `ValidatorByPowerIndex`中删除
- 如果是离开小黑屋则增加记录到 `ValidatorByPowerIndex`

## Delegations

### Delegate

当进行投票的时候,投票人和节点都会受到影响

- 根据投票人的代币和节点的汇率确定投票人的股份
- 从投票人账户上面移除代币
- 新建一个新的投票对象或者已经有一个投票对象
- 将投票份额添加到投票对象中并更新节点对象 
- 将 `delegation.Amount` 从投票账户转移到`BondedPool`或者 `NotBondedPool` `ModuleAccount` 取决于 `validator.Status` 是 `Bonded` 或其他
- 从 `ValidatorByPowerIndex`删除已经存在的记录
- 增加一条新的记录到 `ValidatorByPowerIndex`

### Begin Unbonding

作为取消投票和完成Unbonding状态转换Unbond功能可能会被调用.

- 从委托人里面减去未绑定的份额.
- 如果节点状态是 `Unbonding` 或 `Bonded`将token添加到`UnbondingDelegation` 对象中
- 如果节点状态是`Unbonded` 将token直接转到取币账户中
- 更新投票信息如果投票信息中份额是0则直接删除
- 如果是节点操作者进行操作并且该节点不再有任何份额,则直接关小黑屋
- 更新节点,移除投票人的份额和关联的代币
- 如果节点状态是 `Bonded`, 将价值为未绑定份额的 `Coins`从`BondedPool`转到 `NotBondedPool` `ModuleAccount`
- 如果节点状态是unbonded并且没有任何投票则将该节点删除

### Complete Unbonding

对于尚未完成的取消投票的操作,当冻结时间到了会发生以下操作:

- r将条目从 `UnbondingDelegation` 移除
- 将代币从 `NotBondedPool` `ModuleAccount` 转到投票`Account`中
  
### Begin Redelegation

转投你会影响投票人,源节点和目标节点

- 在源节点执行一个`unbond`委托来检索待解绑的份额的价值
- 使用解绑的代币, 将他们投票到目标节点
- 如果 源节点的状态是 `Bonded`, 目标节点的状态不是`Bonded`, 
  将投票代币从`BondedPool`转到 `NotBondedPool` `ModuleAccount`
- 另外,如果源节点的状态不是 `Bonded`, 但是目标节点的状态是 `Bonded`, 将投票代币从`NotBondedPool` 转移到 `BondedPool` `ModuleAccount`
- 将代币金额记录在`Redelegation`相关条目中

### Complete Redelegation

当转投结束的时候会做一下操作

- 从`Redelegation`对象中删除对应条目

## Slashing

### Slash Validator

当一个节点被惩罚,会发生一下操作:

- 计算总的 `slashAmount`  = `slashFactor` (一个链上参数) * `TokensFromConsensusPower`,违规时节点上投票的总的代币数量
-节点的每个unbonding操作和基于该节点的转投都通过initialBalance使用`slashFactor`来进行削减
- 转投和取消委托将在总削减金额中减去
- 剩余削减金额将会从根据节点状态节点的 `BondedPool` 或者 `NonBondedPool`减掉 ,这将减少代币的总量

### Slash Unbonding Delegation

当Unbonding节点被惩罚时,那些投票将在被惩罚的时候开始取消投票,节点的每个投票都会被`slashFactor`锁定,削减的金额是根据`InitialBalance`进行计算得到的,并有一个上限,以防出现负数.已完成的取消投票金额不会被削减.

### Slash Redelegation

当节点被惩罚是,这个块上所有从该节点的转投也会受到惩罚,转投的金额会由`slashFactor`进行削减,削减的金额会在`InitialBalance`中计算并设置上限以防出现负数,已经完成的转投不会被削减.