# BeginBlock

## Liveness Tracking

在每个块的开头，我们为每个节点更新` ValidatorSigningInfo`，并检查它们是否在滑动窗口上超过了活跃度阈值。 该滑动窗口由` SignedBlocksWindow`定义，并且该窗口中的索引由在验证器的` ValidatorSigningInfo`中找到的` IndexOffset`确定。 对于每个处理的块，无论验证器是否签名，` IndexOffset`都会增加。 确定索引后，将相应更新` MissedBlocksBitArray`和` MissedBlocksCounter`。

最后，为了确定验证程序是否超过活跃度阈值，我们获取丢失的最大块数maxMissed，即SignedBlocksWindow-（MinSignedPerWindow * SignedBlocksWindow）和确定活跃度的最小高度， minHeight。 如果当前块大于" minHeight"，并且验证器的" MissedBlocksCounter"大于" maxMissed"，它们将被" SlashFractionDowntime"削减，将被" DowntimeJailDuration"监禁，并重置以下值：" MissedBlocksBitArray" ，" MissedBlocksCounter"和" IndexOffset"。

__注意__：真实的斜线**不会**导致墓碑化

```go
height := block.Height

for vote in block.LastCommitInfo.Votes {
  signInfo := GetValidatorSigningInfo(vote.Validator.Address)

  // This is a relative index, so we counts blocks the validator SHOULD have
  // signed. We use the 0-value default signing info if not present, except for
  // start height.
  index := signInfo.IndexOffset % SignedBlocksWindow()
  signInfo.IndexOffset++

  // Update MissedBlocksBitArray and MissedBlocksCounter. The MissedBlocksCounter
  // just tracks the sum of MissedBlocksBitArray. That way we avoid needing to
  // read/write the whole array each time.
  missedPrevious := GetValidatorMissedBlockBitArray(vote.Validator.Address, index)
  missed := !signed

  switch {
  case !missedPrevious && missed:
    // array index has changed from not missed to missed, increment counter
    SetValidatorMissedBlockBitArray(vote.Validator.Address, index, true)
    signInfo.MissedBlocksCounter++

  case missedPrevious && !missed:
    // array index has changed from missed to not missed, decrement counter
    SetValidatorMissedBlockBitArray(vote.Validator.Address, index, false)
    signInfo.MissedBlocksCounter--

  default:
    // array index at this index has not changed; no need to update counter
  }

  if missed {
    // emit events...
  }

  minHeight := signInfo.StartHeight + SignedBlocksWindow()
  maxMissed := SignedBlocksWindow() - MinSignedPerWindow()

  // If we are past the minimum height and the validator has missed too many
  // jail and slash them.
  if height > minHeight && signInfo.MissedBlocksCounter > maxMissed {
    validator := ValidatorByConsAddr(vote.Validator.Address)

    // emit events...

    // We need to retrieve the stake distribution which signed the block, so we
    // subtract ValidatorUpdateDelay from the block height, and subtract an
    // additional 1 since this is the LastCommit.
    //
    // Note, that this CAN result in a negative "distributionHeight" up to
    // -ValidatorUpdateDelay-1, i.e. at the end of the pre-genesis block (none) = at the beginning of the genesis block.
    // That's fine since this is just used to filter unbonding delegations & redelegations.
    distributionHeight := height - sdk.ValidatorUpdateDelay - 1

    Slash(vote.Validator.Address, distributionHeight, vote.Validator.Power, SlashFractionDowntime())
    Jail(vote.Validator.Address)

    signInfo.JailedUntil = block.Time.Add(DowntimeJailDuration())

    // We need to reset the counter & array so that the validator won't be
    // immediately slashed for downtime upon rebonding.
    signInfo.MissedBlocksCounter = 0
    signInfo.IndexOffset = 0
    ClearValidatorMissedBlockBitArray(vote.Validator.Address)
  }

  SetValidatorSigningInfo(vote.Validator.Address, signInfo)
}
```