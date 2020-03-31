# Messages

在本节中，我们描述了" slashing"模块的消息处理。

## Unjail

如果验证器由于停机而自动取消绑定，并且希望重新联机并可能重新加入绑定集，则它必须发送`TxUnjail`：

```
type TxUnjail struct {
    ValidatorAddr sdk.AccAddress
}

handleMsgUnjail(tx TxUnjail)

    validator = getValidator(tx.ValidatorAddr)
    if validator == nil
      fail with "No validator found"

    if !validator.Jailed
      fail with "Validator not jailed, cannot unjail"

    info = GetValidatorSigningInfo(operator)
    if info.Tombstoned
      fail with "Tombstoned validator cannot be unjailed"
    if block time < info.JailedUntil
      fail with "Validator still jailed, cannot unjail until period has expired"

    validator.Jailed = false
    setValidator(validator)

    return
```

如果验证者有足够的赌注可以放在顶部n = aximumBondedValidators中，它们将被自动重新绑定，所有仍委派给验证者的代表也将被重新绑定，并开始再次收集准备金和奖励。