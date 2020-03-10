# 本文档学习cosmos上面领取投票分红的相关逻辑

领取分红的函数withdrawDelegationRewards,被相应领取分红的消息handleMsgWithdrawDelegatorReward和BeforeDelegationSharesModified调用.

+ handleMsgWithdrawDelegatorReward处理主动领取分红的动作,调用 WithdrawDelegationRewards
+ BeforeDelegationSharesModified 当投票数额变动的时候自动调用,调用withdrawDelegationRewards

## handleMsgWithdrawDelegatorReward

handleMsgWithdrawDelegatorReward 直接调用WithdrawDelegationRewards,返回相应处理Event的Result

## BeforeDelegationSharesModified

BeforeDelegationSharesModified 是在投票数变化的时候进行调用,调用withdrawDelegationRewards后返回

## WithdrawDelegationRewards

WithdrawDelegationRewards函数调用withdrawDelegationRewards以后会调用initializeDelegation重新初始化投票信息,以便下次领取投票分红的时候能正确计算出可以领取的分红.

## withdrawDelegationRewards

withdrawDelegationRewards是领取分红的主要函数,处理计算分红,领取分红等功能

```delegation.go

	endingPeriod := k.incrementValidatorPeriod(ctx, val)
	rewardsRaw := k.calculateDelegationRewards(ctx, val, del, endingPeriod)
	outstanding := k.GetValidatorOutstandingRewardsCoins(ctx, del.GetValidatorAddr())

        rewards := rewardsRaw.Intersect(outstanding)
        coins, remainder := rewards.TruncateDecimal()
         withdrawAddr := k.GetDelegatorWithdrawAddr(ctx, del.GetDelegatorAddr())
        err := k.supplyKeeper.SendCoinsFromModuleToAccount(ctx, types.ModuleName, withdrawAddr, coins)

    k.SetValidatorOutstandingRewards(ctx, del.GetValidatorAddr(), types.ValidatorOutstandingRewards{Rewards: outstanding.Sub(rewards)})
    feePool := k.GetFeePool(ctx)
    feePool.CommunityPool = feePool.CommunityPool.Add(remainder...)
    k.SetFeePool(ctx, feePool)

    startingInfo := k.GetDelegatorStartingInfo(ctx, del.GetValidatorAddr(), del.GetDelegatorAddr())
    startingPeriod := startingInfo.PreviousPeriod
    k.decrementReferenceCount(ctx, del.GetValidatorAddr(), startingPeriod)
    k.DeleteDelegatorStartingInfo(ctx, del.GetValidatorAddr(), del.GetDelegatorAddr())

```
下面简单分下下代码

### withdrawDelegationRewards-第一部分

+ endingPeriod := k.incrementValidatorPeriod(ctx, val)  调用incrementValidatorPeriod函数,将BP上面的当前奖励归纳到历史奖励中,因为所有奖励都是从历史奖励中领取的
+ rewardsRaw := k.calculateDelegationRewards(ctx, val, del, endingPeriod) 计算分红
+ outstanding := k.GetValidatorOutstandingRewardsCoins(ctx, del.GetValidatorAddr())  获取节点尚未分出去的奖励

incrementValidatorPeriod  在文档block_reward_transfer有详细解释,这里不再赘述,现在主要分析下待领奖励的计算calculateDelegationRewards

## calculateDelegationRewards

```delegation.go

startingInfo := k.GetDelegatorStartingInfo(ctx, del.GetValidatorAddr(), del.GetDelegatorAddr())
stake := startingInfo.Stake

    if endingHeight > startingHeight {
        k.IterateValidatorSlashEventsBetween(ctx, del.GetValidatorAddr(), startingHeight, endingHeight,
            func(height uint64, event types.ValidatorSlashEvent) (stop bool) {
                endingPeriod := event.ValidatorPeriod
                if endingPeriod > startingPeriod {
                    rewards = rewards.Add(k.calculateDelegationRewardsBetween(ctx, val, startingPeriod, endingPeriod, stake)...)

                    // Note: It is necessary to truncate so we don't allow withdrawing
                    // more rewards than owed.
                    stake = stake.MulTruncate(sdk.OneDec().Sub(event.Fraction))
                    startingPeriod = endingPeriod
                }
                return false
            },
        )
    }

    currentStake := val.TokensFromShares(del.GetShares())
    if stake.GT(currentStake) {
            marginOfErr := sdk.SmallestDec().MulInt64(3)
            if stake.LTE(currentStake.Add(marginOfErr)) {
                stake = currentStake
            } else {
                panic(fmt.Sprintf("calculated final stake for delegator %s greater than current stake"+
                    "\n\tfinal stake:\t%s"+
                    "\n\tcurrent stake:\t%s",
                    del.GetDelegatorAddr(), stake, currentStake))
            }
    }

    rewards = rewards.Add(k.calculateDelegationRewardsBetween(ctx, val, startingPeriod, endingPeriod, stake)...)
    return rewards

```

这里分成4个部分来分析

### calculateDelegationRewards-第一部分

+ startingInfo := k.GetDelegatorStartingInfo(ctx, del.GetValidatorAddr(), del.GetDelegatorAddr())
+ stake := startingInfo.Stake

这里获取2个重要参数 startingInfo 记录可以领取的起始投票的信息, stake单独列出来,这个参数是该领取人的股权

### calculateDelegationRewards-第二部分

```delegation.go
    if endingHeight > startingHeight {
        k.IterateValidatorSlashEventsBetween(ctx, del.GetValidatorAddr(), startingHeight, endingHeight,
            func(height uint64, event types.ValidatorSlashEvent) (stop bool) {
                endingPeriod := event.ValidatorPeriod
                if endingPeriod > startingPeriod {
                    rewards = rewards.Add(k.calculateDelegationRewardsBetween(ctx, val, startingPeriod, endingPeriod, stake)...)

                    // Note: It is necessary to truncate so we don't allow withdrawing
                    // more rewards than owed.
                    stake = stake.MulTruncate(sdk.OneDec().Sub(event.Fraction))
                    startingPeriod = endingPeriod
                }
                return false
            },
        )
    }
```

这里面调用了函数  IterateValidatorSlashEventsBetween,这个函数的实现:

```store.go

// iterate over slash events between heights, inclusive
func (k Keeper) IterateValidatorSlashEventsBetween(ctx sdk.Context, val sdk.ValAddress, startingHeight uint64, endingHeight uint64,
    handler func(height uint64, event types.ValidatorSlashEvent) (stop bool)) {
    store := ctx.KVStore(k.storeKey)
    iter := store.Iterator(
        types.GetValidatorSlashEventKeyPrefix(val, startingHeight),
        types.GetValidatorSlashEventKeyPrefix(val, endingHeight+1),
    )
    defer iter.Close()
    for ; iter.Valid(); iter.Next() {
        var event types.ValidatorSlashEvent
        k.cdc.MustUnmarshalBinaryLengthPrefixed(iter.Value(), &event)
        _, height := types.GetValidatorSlashEventAddressHeight(iter.Key())
        if handler(height, event) {
            break
        }
    }
}

```

这个函数的功能是根据高度获取到两个高度中所有关于惩罚相关的事件,根据相关的事件分别调用传进来的handler函数.

handler函数的功能是计算startingPeriod传入的event.Period之间投票人获取的分红,并更新对应的Stake,这里使用循环调用,计算出此区间所有分红的总和

#### calculateDelegationRewardsBetween 函数计算两个区间中获得的分红

```delegation.go

// calculate the rewards accrued by a delegation between two periods
func (k Keeper) calculateDelegationRewardsBetween(ctx sdk.Context, val exported.ValidatorI,
    startingPeriod, endingPeriod uint64, stake sdk.Dec) (rewards sdk.DecCoins) {
    // sanity check
    if startingPeriod > endingPeriod {
        panic("startingPeriod cannot be greater than endingPeriod")
    }

    // sanity check
    if stake.IsNegative() {
        panic("stake should not be negative")
    }

    // return staking * (ending - starting)
    starting := k.GetValidatorHistoricalRewards(ctx, val.GetOperator(), startingPeriod)
    ending := k.GetValidatorHistoricalRewards(ctx, val.GetOperator(), endingPeriod)
    difference := ending.CumulativeRewardRatio.Sub(starting.CumulativeRewardRatio)
    if difference.IsAnyNegative() {
        panic("negative rewards should not be possible")
    }
    // note: necessary to truncate so we don't allow withdrawing more rewards than owed
    rewards = difference.MulDecTruncate(stake)
    return
}

```

通过获取startingPeriod的总分红和endingPeriod的总分红,计算出两个分红的差值就是在这个区间内获取的分红,然后MulDecTruncate对应的股份,计算出投票人应该有的分红

### calculateDelegationRewards-第三部分

```delegation.go

    currentStake := val.TokensFromShares(del.GetShares())
    if stake.GT(currentStake) {
            marginOfErr := sdk.SmallestDec().MulInt64(3)
            if stake.LTE(currentStake.Add(marginOfErr)) {
                stake = currentStake
            } else {
                panic(fmt.Sprintf("calculated final stake for delegator %s greater than current stake"+
                    "\n\tfinal stake:\t%s"+
                    "\n\tcurrent stake:\t%s",
                    del.GetDelegatorAddr(), stake, currentStake))
            }
    }

```

这部分的功能是校验计算得到的stake是否大于currentStake,主要是校验发现是否计算出错.

### calculateDelegationRewards-第四部分

+ rewards = rewards.Add(k.calculateDelegationRewardsBetween(ctx, val, startingPeriod, endingPeriod, stake)...)

这里计算最后一个周期的分红,因为这个周期是在这个函数刚开始调用incrementValidatorPeriod的时候总结的,所以单独计算

### withdrawDelegationRewards-第二部分

```delegation.go

        rewards := rewardsRaw.Intersect(outstanding)
        coins, remainder := rewards.TruncateDecimal()
         withdrawAddr := k.GetDelegatorWithdrawAddr(ctx, del.GetDelegatorAddr())
        err := k.supplyKeeper.SendCoinsFromModuleToAccount(ctx, types.ModuleName, withdrawAddr, coins)

```

+ rewards := rewardsRaw.Intersect(outstanding) 这里分红取计算出来的分红和未发放的分红中的最小值
+ coins, remainder := rewards.TruncateDecimal() 这里获取代币的精确部分和截断部分
+ withdrawAddr := k.GetDelegatorWithdrawAddr(ctx, del.GetDelegatorAddr()) 获取接收地址
+  err := k.supplyKeeper.SendCoinsFromModuleToAccount(ctx, types.ModuleName, withdrawAddr, coins) 将分红从对应的分红池写入转入对应的地址中

### withdrawDelegationRewards-第三部分

```delegation.go

    k.SetValidatorOutstandingRewards(ctx, del.GetValidatorAddr(), types.ValidatorOutstandingRewards{Rewards: outstanding.Sub(rewards)})
    feePool := k.GetFeePool(ctx)
    feePool.CommunityPool = feePool.CommunityPool.Add(remainder...)
    k.SetFeePool(ctx, feePool)

```

这里一共2个功能,SetValidatorOutstandingRewards更新未发放分红,将截断部分放置到社区分红池中.

### withdrawDelegationRewards-第四部分

```delegation.go

    startingInfo := k.GetDelegatorStartingInfo(ctx, del.GetValidatorAddr(), del.GetDelegatorAddr())
    startingPeriod := startingInfo.PreviousPeriod
    k.decrementReferenceCount(ctx, del.GetValidatorAddr(), startingPeriod)
    k.DeleteDelegatorStartingInfo(ctx, del.GetValidatorAddr(), del.GetDelegatorAddr())

```

这里对startingInfo进行处理,先减少对应的引用计数,然后把这个startingInfo删除.

## 后记

这个文档简要分析了领取分红的相应代码,唯一不是特别明确的地方就是rewards = difference.MulDecTruncate(stake)  这里把BP分红和投票人股份进行相乘并截断,这里的Stake是否是完全根据百分比得到的股份有待确认.