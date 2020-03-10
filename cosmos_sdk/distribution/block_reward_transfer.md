# 本文档学习当区块开始的时候BP上奖励的转移

因为在分配费用的时候会把费用等相关奖励放置到ValidatorAccumulatedCommission和ValidatorCurrentRewards两个地方,ValidatorAccumulatedCommission是BP的佣金,这里主要研究下ValidatorCurrentRewards中的代币如何转移.

在函数 incrementValidatorPeriod中,这个函数在每次投票改变或者领取分红的时候都会进行调用.

## incrementValidatorPeriod

```validator.go
rewards := k.GetValidatorCurrentRewards(ctx, val.GetOperator())
historical := k.GetValidatorHistoricalRewards(ctx, val.GetOperator(), rewards.Period-1).CumulativeRewardRatio
k.decrementReferenceCount(ctx, val.GetOperator(), rewards.Period-1)
k.SetValidatorHistoricalRewards(ctx, val.GetOperator(), rewards.Period, types.NewValidatorHistoricalRewards(historical.Add(current...), 1))
k.SetValidatorCurrentRewards(ctx, val.GetOperator(), types.NewValidatorCurrentRewards(sdk.DecCoins{}, rewards.Period+1))
```

核心代码就上面几句,功能分别是:

+ 获取CurrentReward上面的奖励
+ 获取CumulativeRewardRatio  根据字面意思是奖励值
+ 减少历史奖励值的参考计数  这个计数在历史奖励最初创建的时候增加,这个时候减少应该是这个历史奖励的归纳只能归纳一次
+ 设置历史奖励,也就是把当前奖励归为最新的历史奖励
+ 将当前奖励设置成0

综上,这个函数的意义就是将当前奖励归为历史奖励,历史奖励其实是累加的,也就是说后面的记录一定是加上前面记录以后的一个值

