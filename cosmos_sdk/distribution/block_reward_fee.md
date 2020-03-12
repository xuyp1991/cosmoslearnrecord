# cosmos手续费的分配

本文档通过阅读代码的形式简单总结下Cosmos出块奖励的分配

## 代码入口

分配奖励的代码入口是在distribution模块下面abci.go的BeginBlocker函数,该函数调用AllocateTokens进行奖励分配,分配给上一个出块的节点.也就是说第二个区块的开始才是第一个区块的结束,第二个区块要总结第一个区块的相关信息并给出第一个区块的奖励.

## BeginBlocker

接下来看一下BeginBlocker的相关内容

+ 1.BeginBlocker首先总结上一个区块生产者的投票信息,previousTotalPower存储总的投票数,sumPreviousPrecommitPower存储对上一个区块进行签名的投票数.
+ 2.获取这些值以后调用AllocateTokens进行奖励的分配.
+ 3.奖励分配以后将自己设置为上一个出块人,以便下一个区块生产的时候给自己奖励.

```abci.go
    for _, voteInfo := range req.LastCommitInfo.GetVotes() {
        previousTotalPower += voteInfo.Validator.Power
        if voteInfo.SignedLastBlock {
            sumPreviousPrecommitPower += voteInfo.Validator.Power
        }
    }

    if ctx.BlockHeight() > 1 {
        previousProposer := k.GetPreviousProposerConsAddr(ctx)
        k.AllocateTokens(ctx, sumPreviousPrecommitPower, previousTotalPower, previousProposer, req.LastCommitInfo.GetVotes())
    }

    consAddr := sdk.ConsAddress(req.Header.ProposerAddress)
    k.SetPreviousProposerConsAddr(ctx, consAddr)

```

## AllocateTokens

AllocateTokens函数是分配奖励的具体函数,首先看一下它的入参

### 入参

+ sumPreviousPrecommitPower  进行签名的投票数
+ totalPreviousPower        总的投票数
+ previousProposer            BP地址
+ previousVotes                    该BP上面所有的投票信息

### 功能概述

+ 获取待分配的token,这里看注释是获取手续费
+ 分配奖励给BP
+ 分配奖励给投票人
+ 剩余的钱分给社区

### 获取待分配的奖励

```allocation.go
    feeCollector := k.supplyKeeper.GetModuleAccount(ctx, k.feeCollectorName)
    feesCollectedInt := k.bankKeeper.GetAllBalances(ctx, feeCollector.GetAddress())
    feesCollected := sdk.NewDecCoinsFromCoins(feesCollectedInt...)
```

由代码可知从feeCollectorName这个账户的地址上面获取所有的代币,这个代币准确来说是增发的代币,不是 收集费用得到的代币

### 分配奖励给BP

```allocation.go
// calculate fraction votes
	previousFractionVotes := sdk.NewDec(sumPreviousPrecommitPower).Quo(sdk.NewDec(totalPreviousPower))

	// calculate previous proposer reward	
	baseProposerReward := k.GetBaseProposerReward(ctx)
	bonusProposerReward := k.GetBonusProposerReward(ctx)
	proposerMultiplier := baseProposerReward.Add(bonusProposerReward.MulTruncate(previousFractionVotes))
	proposerReward := feesCollected.MulDecTruncate(proposerMultiplier)

	// pay previous proposer
	remaining := feesCollected
	proposerValidator := k.stakingKeeper.ValidatorByConsAddr(ctx, previousProposer)

	if proposerValidator != nil {
		ctx.EventManager().EmitEvent(
			sdk.NewEvent(
				types.EventTypeProposerReward,
				sdk.NewAttribute(sdk.AttributeKeyAmount, proposerReward.String()),
				sdk.NewAttribute(types.AttributeKeyValidator, proposerValidator.GetOperator().String()),
			),
		)

		k.AllocateTokensToValidator(ctx, proposerValidator, proposerReward)
		remaining = remaining.Sub(proposerReward)//获取分配后的剩余部分
	} 
```

给BP的奖励这里有2个部分,base和bound,奖励的计算是base+bound * sumPreviousPrecommitPower / totalPreviousPower

**base和bound都是比例最终计算的结果也是一个在总奖励中的比例**

通过函数AllocateTokensToValidator把奖励分给BP

### AllocateTokensToValidator

AllocateTokensToValidator函数分配奖励给一个地址,奖励分2个部分到地址的相关存储字段上面,commission和shared,分别代表佣金和当前奖励,将代币部分记录在未分配奖励部分

```allocation.go
    commission := tokens.MulDec(val.GetCommission())
    shared := tokens.Sub(commission)

    currentCommission := k.GetValidatorAccumulatedCommission(ctx, val.GetOperator())
    currentCommission.Commission = currentCommission.Commission.Add(commission...)
    k.SetValidatorAccumulatedCommission(ctx, val.GetOperator(), currentCommission)

     currentRewards := k.GetValidatorCurrentRewards(ctx, val.GetOperator())
    currentRewards.Rewards = currentRewards.Rewards.Add(shared...)
    k.SetValidatorCurrentRewards(ctx, val.GetOperator(), currentRewards)

    outstanding := k.GetValidatorOutstandingRewards(ctx, val.GetOperator())
    outstanding.Rewards = outstanding.Rewards.Add(tokens...)
    k.SetValidatorOutstandingRewards(ctx, val.GetOperator(), outstanding)
```

### 分配奖励给其他BP  bonded validators

```allocation.go
 // calculate fraction allocated to validators计算分配给验证者的分数
    communityTax := k.GetCommunityTax(ctx)
    voteMultiplier := sdk.OneDec().Sub(proposerMultiplier).Sub(communityTax)

    // allocate tokens proportionally to voting power
    // TODO consider parallelizing later, ref https://github.com/cosmos/cosmos-sdk/pull/3099#discussion_r246276376  给验证人分配
    for _, vote := range previousVotes {
        validator := k.stakingKeeper.ValidatorByConsAddr(ctx, vote.Validator.Address)

        // TODO consider microslashing for missing votes.
        // ref https://github.com/cosmos/cosmos-sdk/issues/2525#issuecomment-430838701
        powerFraction := sdk.NewDec(vote.Validator.Power).QuoTruncate(sdk.NewDec(totalPreviousPower))
        reward := feesCollected.MulDecTruncate(voteMultiplier).MulDecTruncate(powerFraction)
        k.AllocateTokensToValidator(ctx, validator, reward)
        remaining = remaining.Sub(reward)
    }
```

这里一个验证人能分配到的奖励是  reward = total_reward * (1-给社区的比例-给BP的比例)*(投票数/总投票数)

通过函数AllocateTokensToValidator将奖金分配给验证人

### 分配奖励给社区

```allocation.go
    feePool.CommunityPool = feePool.CommunityPool.Add(remaining...)
    k.SetFeePool(ctx, feePool)
```

由代码可知,剩下所有未分配的都给了社区

## 后记

由此我们便分析了所有分配的流程,所有的分配只是记账,并没有实质代币的分配,BP的收益有2部分,基础收益和根据投票数产生的收益,给用户的就是根据投票数和总投票数来分配给用户,剩下的部分全部都给社区
