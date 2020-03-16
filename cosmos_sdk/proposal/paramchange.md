# Cosmos参数修改提案

cosmos参数修改提案也是走的正常提案的入口.

## 申请参数修改的提案

```tx.go
    content := paramproposal.NewParameterChangeProposal(proposal.Title, proposal.Description, proposal.Changes.ToParamChanges())
    msg := govtypes.NewMsgSubmitProposal(content, proposal.Deposit, from)
```

和upgrade一样生成1个ParameterChangeProposal的content对象,调用gov的NewMsgSubmitProposal进行提案.

```tx.go
    ParamChangeJSON struct {
        Subspace string          `json:"subspace" yaml:"subspace"`
        Key      string          `json:"key" yaml:"key"`
        Value    json.RawMessage `json:"value" yaml:"value"`
    }
```

关于修改参数的相关结构如上

+ Subspace      模块
+ Key                  键
+ Value              值

## 提案通过以后的处理

params模块提案绑定的处理函数是NewParamChangeProposalHandler

```proposal_handler.go
        case proposal.ParameterChangeProposal:
                return handleParameterChangeProposal(ctx, k, c)
```

该函数最终调用 handleParameterChangeProposal函数

```proposal_handler.go
ss, ok := k.GetSubspace(c.Subspace)
ss.Update(ctx, []byte(c.Key), []byte(c.Value));
```

handleParameterChangeProposal函数最终将值更新到Subspace里面.

## 提案如何使修改的参数生效.

由参数处理相关函数可以知道,参数修改提案最终只是修改了Subspace中的一条记录而已.,如果这个记录修改以后就生效的话,那么其他模块做处理的时候对应参数一定是从相关记录里面取值进行处理的.

由代码可以知道所有模块都有函数GetParams(ctx sdk.Context) (params types.Params)来获取相关参数,这样所有调用这个函数获取的参数都是可以通过paramchange提案进行修改的.

## 总结一下可以修改的参数

### auth

legacy 

```v0_34
    Params struct {
        MaxMemoCharacters      uint64 `json:"max_memo_characters"`
        TxSigLimit             uint64 `json:"tx_sig_limit"`
        TxSizeCostPerByte      uint64 `json:"tx_size_cost_per_byte"`
        SigVerifyCostED25519   uint64 `json:"sig_verify_cost_ed25519"`
        SigVerifyCostSecp256k1 uint64 `json:"sig_verify_cost_secp256k1"`
    }
```

```auth
type Params struct {
    MaxMemoCharacters      uint64 `protobuf:"varint,1,opt,name=max_memo_characters,json=maxMemoCharacters,proto3" json:"max_memo_characters,omitempty" yaml:"max_memo_characters"`
    TxSigLimit             uint64 `protobuf:"varint,2,opt,name=tx_sig_limit,json=txSigLimit,proto3" json:"tx_sig_limit,omitempty" yaml:"tx_sig_limit"`
    TxSizeCostPerByte      uint64 `protobuf:"varint,3,opt,name=tx_size_cost_per_byte,json=txSizeCostPerByte,proto3" json:"tx_size_cost_per_byte,omitempty" yaml:"tx_size_cost_per_byte"`
    SigVerifyCostED25519   uint64 `protobuf:"varint,4,opt,name=sig_verify_cost_ed25519,json=sigVerifyCostEd25519,proto3" json:"sig_verify_cost_ed25519,omitempty" yaml:"sig_verify_cost_ed25519"`
    SigVerifyCostSecp256k1 uint64 `protobuf:"varint,5,opt,name=sig_verify_cost_secp256k1,json=sigVerifyCostSecp256k1,proto3" json:"sig_verify_cost_secp256k1,omitempty" yaml:"sig_verify_cost_secp256k1"`
}
```

### distribution

```distribution
type Params struct {
    CommunityTax        github_com_cosmos_cosmos_sdk_types.Dec `protobuf:"bytes,1,opt,name=community_tax,json=communityTax,proto3,customtype=github.com/cosmos/cosmos-sdk/types.Dec" json:"community_tax" yaml:"community_tax"`
    BaseProposerReward  github_com_cosmos_cosmos_sdk_types.Dec `protobuf:"bytes,2,opt,name=base_proposer_reward,json=baseProposerReward,proto3,customtype=github.com/cosmos/cosmos-sdk/types.Dec" json:"base_proposer_reward" yaml:"base_proposer_reward"`
    BonusProposerReward github_com_cosmos_cosmos_sdk_types.Dec `protobuf:"bytes,3,opt,name=bonus_proposer_reward,json=bonusProposerReward,proto3,customtype=github.com/cosmos/cosmos-sdk/types.Dec" json:"bonus_proposer_reward" yaml:"bonus_proposer_reward"`
    WithdrawAddrEnabled bool                                   `protobuf:"varint,4,opt,name=withdraw_addr_enabled,json=withdrawAddrEnabled,proto3" json:"withdraw_addr_enabled,omitempty" yaml:"withdraw_addr_enabled"`
}
```

legacy 

```v0_38
    Params struct {
        CommunityTax        sdk.Dec `json:"community_tax" yaml:"community_tax"`
        BaseProposerReward  sdk.Dec `json:"base_proposer_reward" yaml:"base_proposer_reward"`
        BonusProposerReward sdk.Dec `json:"bonus_proposer_reward" yaml:"bonus_proposer_reward"`
        WithdrawAddrEnabled bool    `json:"withdraw_addr_enabled" yaml:"withdraw_addr_enabled"`
    }
```

### evidence

```distribution
type Params struct {
    MaxEvidenceAge time.Duration `protobuf:"bytes,1,opt,name=max_evidence_age,json=maxEvidenceAge,proto3,stdduration" json:"max_evidence_age" yaml:"max_evidence_age"`
}
```

### gov

```gov
// Params returns all of the governance params
type Params struct {
    VotingParams  VotingParams  `json:"voting_params" yaml:"voting_params"`
    TallyParams   TallyParams   `json:"tally_params" yaml:"tally_params"`
    DepositParams DepositParams `json:"deposit_params" yaml:"deposit_parmas"`
}
```

### mint

```mint
// mint parameters
type Params struct {
    // type of coin to mint
    MintDenom string `protobuf:"bytes,1,opt,name=mint_denom,json=mintDenom,proto3" json:"mint_denom,omitempty"`
    // maximum annual change in inflation rate
    InflationRateChange github_com_cosmos_cosmos_sdk_types.Dec `protobuf:"bytes,2,opt,name=inflation_rate_change,json=inflationRateChange,proto3,customtype=github.com/cosmos/cosmos-sdk/types.Dec" json:"inflation_rate_change" yaml:"inflation_rate_change"`
    // maximum inflation rate
    InflationMax github_com_cosmos_cosmos_sdk_types.Dec `protobuf:"bytes,3,opt,name=inflation_max,json=inflationMax,proto3,customtype=github.com/cosmos/cosmos-sdk/types.Dec" json:"inflation_max" yaml:"inflation_max"`
    // minimum inflation rate
    InflationMin github_com_cosmos_cosmos_sdk_types.Dec `protobuf:"bytes,4,opt,name=inflation_min,json=inflationMin,proto3,customtype=github.com/cosmos/cosmos-sdk/types.Dec" json:"inflation_min" yaml:"inflation_min"`
    // goal of percent bonded atoms
    GoalBonded github_com_cosmos_cosmos_sdk_types.Dec `protobuf:"bytes,5,opt,name=goal_bonded,json=goalBonded,proto3,customtype=github.com/cosmos/cosmos-sdk/types.Dec" json:"goal_bonded" yaml:"goal_bonded"`
    // expected blocks per year
    BlocksPerYear uint64 `protobuf:"varint,6,opt,name=blocks_per_year,json=blocksPerYear,proto3" json:"blocks_per_year,omitempty" yaml:"blocks_per_year"`
}
```

### simulations

```simulations
// Params define the parameters necessary for running the simulations
type Params struct {
    PastEvidenceFraction      float64
    NumKeys                   int
    EvidenceFraction          float64
    InitialLivenessWeightings []int
    LivenessTransitionMatrix  TransitionMatrix
    BlockSizeTransitionMatrix TransitionMatrix
}
```

### slashing

```slashing
type Params struct {
    SignedBlocksWindow      int64         `json:"signed_blocks_window" yaml:"signed_blocks_window"`
    MinSignedPerWindow      sdk.Dec       `json:"min_signed_per_window" yaml:"min_signed_per_window"`
    DowntimeJailDuration    time.Duration `json:"downtime_jail_duration" yaml:"downtime_jail_duration"`
    SlashFractionDoubleSign sdk.Dec       `json:"slash_fraction_double_sign" yaml:"slash_fraction_double_sign"`
    SlashFractionDowntime   sdk.Dec       `json:"slash_fraction_downtime" yaml:"slash_fraction_downtime"`
}
```

### staking

legacy  v0_34

```legacy
Params struct {
    UnbondingTime time.Duration `json:"unbonding_time"`
    MaxValidators uint16        `json:"max_validators"`
    MaxEntries    uint16        `json:"max_entries"`
    BondDenom     string        `json:"bond_denom"`
}
```

```staking
type Params struct {
    UnbondingTime     time.Duration `protobuf:"bytes,1,opt,name=unbonding_time,json=unbondingTime,proto3,stdduration" json:"unbonding_time" yaml:"unbonding_time"`
    MaxValidators     uint32        `protobuf:"varint,2,opt,name=max_validators,json=maxValidators,proto3" json:"max_validators,omitempty" yaml:"max_validators"`
    MaxEntries        uint32        `protobuf:"varint,3,opt,name=max_entries,json=maxEntries,proto3" json:"max_entries,omitempty" yaml:"max_entries"`
    HistoricalEntries uint32        `protobuf:"varint,4,opt,name=historical_entries,json=historicalEntries,proto3" json:"historical_entries,omitempty" yaml:"historical_entries"`
    BondDenom         string        `protobuf:"bytes,5,opt,name=bond_denom,json=bondDenom,proto3" json:"bond_denom,omitempty" yaml:"bond_denom"`
}
```