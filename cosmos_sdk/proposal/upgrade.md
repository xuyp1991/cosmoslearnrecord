# 本文学习cosmos关于升级的问题

在Cosmos中升级也是一个单独的提案,所以升级相关指定会走提案的相关处理.

## 申请系统升级的提案命令

```tx.go
msg := gov.NewMsgSubmitProposal(content, deposit, from)
```

在构造msg的时候调用的是gov.NewMsgSubmitProposal来进行提案.

## 提案通过的时候是如何和TEXT提案进行不同的处理的

```app.go
    // register the proposal types
    govRouter := gov.NewRouter()
    govRouter.AddRoute(gov.RouterKey, gov.ProposalHandler).
        AddRoute(paramproposal.RouterKey, params.NewParamChangeProposalHandler(app.ParamsKeeper)).
        AddRoute(distr.RouterKey, distr.NewCommunityPoolSpendProposalHandler(app.DistrKeeper)).
        AddRoute(upgrade.RouterKey, upgrade.NewSoftwareUpgradeProposalHandler(app.UpgradeKeeper))
```

通过`AddRoute(upgrade.RouterKey, upgrade.NewSoftwareUpgradeProposalHandler(app.UpgradeKeeper))`　将NewSoftwareUpgradeProposalHandler绑定到upgrade.RouterKey上面

在构造content的时候,contet直接返回的是一个SoftwareUpgradeProposal对象

```tx.go
content := types.NewSoftwareUpgradeProposal(title, description, plan)
```

然后SoftwareUpgradeProposal的ProposalRoute是upgrade模块的RouterKey.

```proposal.go
func (sup SoftwareUpgradeProposal) ProposalRoute() string  { return RouterKey }
```

最后在通过提案的时候通过proposal.ProposalRoute()绑定的方法进行处理,upgrade调用NewSoftwareUpgradeProposalHandler的返回进行处理.

```abci.go
            handler := keeper.Router().GetRoute(proposal.ProposalRoute())
            err := handler(cacheCtx, proposal.Content)
```

## upgrade提案通过以后的相关处理

```handler.go
func NewSoftwareUpgradeProposalHandler(k Keeper) govtypes.Handler {
    return func(ctx sdk.Context, content govtypes.Content) error {
        switch c := content.(type) {
        case SoftwareUpgradeProposal:
            return handleSoftwareUpgradeProposal(ctx, k, c)

        case CancelSoftwareUpgradeProposal:
            return handleCancelSoftwareUpgradeProposal(ctx, k, c)

        default:
            return sdkerrors.Wrapf(sdkerrors.ErrUnknownRequest, "unrecognized software upgrade proposal content type: %T", c)
        }
    }
}
```

根据函数可以知道 NewSoftwareUpgradeProposalHandler 最终返回了一个临时函数,临时函数中根据提案类型的不同最终进行了不同的处理,升级调用handleSoftwareUpgradeProposal,取消升级调用handleCancelSoftwareUpgradeProposal

handleSoftwareUpgradeProposal最终会调用ScheduleUpgrade函数

```keeper.go
    bz := k.cdc.MustMarshalBinaryBare(&plan)
    store := ctx.KVStore(k.storeKey)
    store.Set(types.PlanKey(), bz)
```

ScheduleUpgrade函数将对应升级的plan写入store里面

handleCancelSoftwareUpgradeProposal最终调用ClearUpgradePlan函数

```keeper.go
    store := ctx.KVStore(k.storeKey)
    store.Delete(types.PlanKey())
```

ClearUpgradePlan最终就是把对应的升级plan删除

## 关于升级计划的执行

升级相关程序是在Upgrade模块的BeginBlocker函数中

```abci.go
    plan, found := k.GetUpgradePlan(ctx)
	if plan.ShouldExecute(ctx) {
        	if k.IsSkipHeight(ctx.BlockHeight()) {
                skipUpgradeMsg := fmt.Sprintf("UPGRADE \"%s\" SKIPPED at %d: %s", plan.Name, plan.Height, plan.Info)
                ctx.Logger().Info(skipUpgradeMsg)

                // Clear the upgrade plan at current height
                k.ClearUpgradePlan(ctx)
                return
            }

            if !k.HasHandler(plan.Name) {
                upgradeMsg := fmt.Sprintf("UPGRADE \"%s\" NEEDED at %s: %s", plan.Name, plan.DueAt(), plan.Info)
                // We don't have an upgrade handler for this upgrade name, meaning this software is out of date so shutdown
                ctx.Logger().Error(upgradeMsg)
                panic(upgradeMsg)
            }
            // We have an upgrade handler for this upgrade name, so apply the upgrade
            ctx.Logger().Info(fmt.Sprintf("applying upgrade \"%s\" at %s", plan.Name, plan.DueAt()))
            ctx = ctx.WithBlockGasMeter(sdk.NewInfiniteGasMeter())
            k.ApplyUpgrade(ctx, plan)
            return
    }

```

+ ShouldExecute 通过判断当前块高度或时间有没有打到升级计划设置的时间或者快高度来确定是否要升级.
+ IsSkipHeight 判断当前块高度是否是终止升级块高度,如果是,则删除升级计划
+ HasHandler判断是否有和升级计划对应的升级函数
+ ApplyUpgrade 执行升级,清除升级计划,将升级计划设置为结束

## 后记

COSMOS链上进行升级的相关内容都总结完毕,有2个疑点

+ 升级计划对应的函数是如何设置上去的
+ 终止升级块高度是如何设置上去的.
