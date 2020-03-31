# Hooks

在本节中，我们描述了"钩子"-削减其他事件发生时运行的模块代码。

## Validator Bonded

首次成功绑定新的验证器后，我们为当前绑定的验证器创建一个新的` ValidatorSigningInfo`结构，即当前块的" StartHeight"。

```go
onValidatorBonded(address sdk.ValAddress)

  signingInfo, found = GetValidatorSigningInfo(address)
  if !found {
    signingInfo = ValidatorSigningInfo {
      StartHeight         : CurrentHeight,
      IndexOffset         : 0,
      JailedUntil         : time.Unix(0, 0),
      Tombstone           : false,
      MissedBloskCounter  : 0
    }
    setValidatorSigningInfo(signingInfo)
  }
  
  return
```