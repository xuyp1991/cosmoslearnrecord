# State

## Signing Info (Liveness)

每个块都包含验证节点对前一个块的一组预提交，成为Tendermint的`LastCommitInfo`，一个`LastCommitInfo`是不可逆的只需要它包含2/3的总投票权的预提交（请看 [TODO](https://github.com/cosmos/cosmos-sdk/issues/967)）

节点可能因为被关小黑屋而没有对块进行签名，从而导致被惩罚或者被设置为unbonded。

通过ValidatorSigningInfo跟踪有关验证者活跃度的信息，并在商店中对其进行索引，如下所示：

- ValidatorSigningInfo: ` 0x01 | ConsAddress -> amino(valSigningInfo)`
- MissedBlocksBitArray: ` 0x02 | ConsAddress | LittleEndianUint64(signArrayIndex) -> VarInt(didMiss)`

基于第一个映射我们可以根据节点的共识地址很轻松的找到节点的签名信息。
第二个映射是大小是`SignedBlocksWindow`的数组，它告诉我们节点是否错过了数组中给定索引的块，索引为小尾数uint64

结果是一个varint，取值为0或1，其中0表示验证者没有丢失（未签名）相应的块，而1表示他们错过了该块（未签名）。 。

注意，`MissedBlocksBitArray`并未预先明确初始化。 随着新绑定的节点的第一个SignedBlocksWindow块的进行，将添加密钥。 " SignedBlocksWindow"参数定义了用于跟踪节点活跃度的滑动窗口的大小（块数）。

用于跟踪节点活动性的存储信息如下：

```go
type ValidatorSigningInfo struct {
    Address             sdk.ConsAddress
    StartHeight         int64
    IndexOffset         int64
    JailedUntil         time.Time
    Tombstoned          bool
    MissedBlocksCounter int64
}
```

说明:

- __Address__: 节点的共识地址.
- __StartHeight__: 候选节点成为验证节点的高度
- __IndexOffset__: 索引，每次节点并且可能签署或者未签署预提交时，索引都会增加，这与SignedBlocksWindow参数的结合确定了MissedBlocksBitArray中的索引。
- __JailedUntil__: 确认程序因活动中断而关小黑屋的时间。
- __Tombstoned__:验证程序的描述是否已墓碑化。 一旦验证者做出模棱两可的设置，或针对任何其他配置的不当行为，将对其进行设置。
- __MissedBlocksCounter__: 保留一个计数器以避免不必要的数组读取。 注意，" Sum（MissedBlocksBitArray）"始终等于" MissedBlocksCounter"。
