# End-Block

每个abci结束块调用，更新队列和节点设置修改的执行。

## Validator Set Changes

在此过程中，通过每个块末尾运行的状态转换器来更新节点投票集。作为此过程的一部分，所有更新的出块节点都会返回给Tendermint，以包含在Tendermint的出块节点中，所有出块节点负责验证Tendermint消息。操作如下：

- 新的出块节点集将从索引ValidatorsByPower中取参数`params.MaxValidators` 定义的个数。
- 之前的出块节点集会和老的出块节点集进行比较：
  - 离开的节点状态修改为 unbonding 它们的代币 `Tokens` 从`BondedPool` 转到 `NotBondedPool` `ModuleAccount`
  - 新的节点状态修改为bonded 它们的代币 `Tokens`从  `NotBondedPool` 转到 `BondedPool` `ModuleAccount`

在任何情况下，所有离开或者加入出块节点集都会产生一条消息，该消息将传回Tendermint

## Queues

在这个模块里面，某些状态转换不是瞬时的，而是在一定时间段内进行的。当转换期到了的时候，必须完成某些操作才能完成转换。这些通过在每个块结束是通过扫描队列来进行实现。

### Unbonding Validators

当一个节点被踢出出块节点集(被关小黑屋或者因为没有足够的票)以后，它就开始一个unbonding过程，所有它的投票也将开始unbonding，这时该节点被成为unbonding节点，尽管当这个unbonding期过去以后它会变成一个unbonded 节点

每个块都会检查节点列表去发现成熟的unbonding节点（即完成时间<=当前时间），这时任何一个成熟节点如果没有任何投票的话都会被删除，对于其他仍有投票的成熟的unbonding节点会将其状态修改从 `sdk.Unbonding` 修改为`sdk.Unbonded`

### Unbonding Delegations

通过一下过程完成对所有`UnbondingDelegations`队列中成熟的 `UnbondingDelegations.Entries` 的解绑操作。

- 将余额转到投票者的钱包地址中
- 从`UnbondingDelegation.Entries`中移除成熟的条目
-如果没有条目，将 `UnbondingDelegation` 对象删除

### Redelegations

通过以下操作完成对`Redelegations`对列中所有成熟的 `Redelegation.Entries`的unbonding操作

- 将成熟的条目从 `Redelegation.Entries`中移除
- 如果没有条目，将`Redelegation` 对象删除
