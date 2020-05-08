# staking

staking 模块是kuchain上面的节点管理模块，实现节点的注册，节点信息修改，给节点投票，转投，取消投票等相关功能。

## 节点说明

在区块链中节点是指区块链网络中的计算机，整个区块链是由一个个节点组成的。

staking模块的功能是管理节点，使kuchain可以支持一种高级的权益证明。在kuchain中，节点的权益是根据节点上投的票权决定的，也就是一个节点上面的投票数越多，该节点的权益就越大 。根据节点的状态分为候选节点，出块节点和受惩罚节点。出块节点是支持整个链平稳运行的节点，负责出块以及对块进行签名验证，出块节点的个数是有限制的，初期是33个，kuchain上出块节点根据节点上的票权来进行选择的，也就是票权最高的33个节点是出块节点。候选节点是指票权不是前33个的普通节点，候选节点不参与出块和验证，也不会获得收益。如果出块节点丢块或对块进行双签，出块节点就会受到惩罚，成为受惩罚节点，在惩罚期间受惩罚节点上面的票权不计入总票权，惩罚结束以后需要执行unjail操作恢复候选节点，然后再去竞争出块节点。

staking模块对节点进行管理需要节点首先在staking模块里面注册，接下来就说一下staking模块的功能。

## 功能说明

### 节点注册

节点注册就是将节点信息在staking模块进行注册一下，以便广播到整个链上。

#### 命令说明

```bash
kucli tx kustaking create-validator [validator_name] [flags]
```

+ validator_name  是节点操作员的名字，目前仅支持传入节点操作员的地址。
+ flags里面有几个是必须的。
  + pubkey是节点的出块公钥。
  + from是为此创建节点msg进行签名并支付手续费的操作员的名字。
  + commission-rate是佣金比例。
  + moniker 是监控器名字，这个最终只是会出现在description上面

执行命令以后可以通过命令查询到注册上去的节点信息

```bash
./kucli tx kustaking create-validator kuchain16xlk6uf0ccsuh3zdff4lwpcafh7k6h6d6hzehk\
  --pubkey=$(./kucd tendermint show-validator --home /home/xuyapeng/go_workspace/src/github.com/KuChain-io/data/kucddata1) \
  --moniker="secode test" \
  --chain-id=testing \
  --from=alice \
  --commission-rate="0.10"


 ./kucli query kustaking validator kuchain16xlk6uf0ccsuh3zdff4lwpcafh7k6h6d6hzehk
{
  "operator_address": "kuchain16xlk6uf0ccsuh3zdff4lwpcafh7k6h6d6hzehk",
  "consensus_pubkey": "kuchainvalconspub1zcjduepqlfcd3vr8evpxf2c4mm9w5m737n6nxj9nn5zl87kn8zkyr8pf6q9swyh04k",
  "status": 1,
  "tokens": "0",
  "delegator_shares": "0.000000000000000000",
  "description": {
    "moniker": "secode test"
  },
  "unbonding_time": "1970-01-01T00:00:00Z",
  "commission": {
    "commission_rates": {
      "rate": "0.100000000000000000",
      "max_rate": "1.000000000000000000",
      "max_change_rate": "1.000000000000000000"
    },
    "update_time": "2020-05-08T07:57:55.539370131Z"
  },
  "min_self_delegation": "1"
}

```

#### 消息说明

```go
type MsgCreateValidator struct {
	Description      Description     
	CommissionRates  github_com_cosmos_cosmos_sdk_types.Dec 
	ValidatorAddress types.AccountID               
	DelegatorAddress types.AccountID               
	Pubkey           string                        
}
```

如果出现以下情况，此消息将失败：

- 已经使用该操作员账户注册了另一个节点
- 已经使用该节点公钥注册了另一个节点
- 抵押代币不是系统主代币
- 佣金参数不符合规定:
  - 初始 `Rate` 是负数或者大于 `1`
- 描述字段太长

这个消息在合适的索引处创建并存储`Validator`对象.

### 修改节点

修改节点用于修改节点的description和rate，也就是描述和佣金比例

#### 命令说明

```bash
kucli tx kustaking edit-validator [validtor_operator_address] [flags]
```

+ validtor_operator_address 就是创建节点的时候输入的validator_name，可以在查询节点信息的时候查询到
+ flag我们这边只修改2个参数
  + commission-rate是佣金比例。
  + moniker 是监控器名字，这个最终只是会出现在description上面

执行命令以后可以通过命令查询到注册上去的节点信息

```bash
./kucli tx kustaking  edit-validator  kuchain16xlk6uf0ccsuh3zdff4lwpcafh7k6h6d6hzehk   --commission-rate="0.20" --moniker="modify test"   --from=alice
```

**节点信息24小时只允许修改一次**

#### 消息说明

```go
type MsgEditValidator struct {
	Description      Description    
	ValidatorAddress types.AccountID
	CommissionRate *github_com_cosmos_cosmos_sdk_types.Dec 
}
```

如果出现以下情况，此消息将失败：

- `CommissionRate` 小于0或者大于 `1`
- `CommissionRate` 已经在24小时内更新过了
- 描述字段太长

这个消息将会存储更新过后的`Validator`对象

### 投票

投票就是将自己的代币抵押给节点，节点获得票权，如果节点是出块节点，投票人可以获得收益，如果节点被惩罚，投票人的抵押会受到损失。

#### 命令说明

```bash
kucli tx kustaking delegate [validator-addr] [amount] [flags]
```

+ validator-addr 就是创建节点的时候输入的validator_name，可以在查询节点信息的时候查询到
+ amount 是投票的代币
+ flags 我们这边只使用到一个From
    + from 为这个msg签名并支付代币的账户

```bash
./kucli tx kustaking delegate kuchain16xlk6uf0ccsuh3zdff4lwpcafh7k6h6d6hzehk 40000000stake --from jack

./kucli query kustaking delegations kuchain19674cxa9s6wl77scgj0nh445s3eqstwtw3g4nu
[
  {
    "delegator_address": "kuchain19674cxa9s6wl77scgj0nh445s3eqstwtw3g4nu",
    "validator_address": "kuchain16xlk6uf0ccsuh3zdff4lwpcafh7k6h6d6hzehk",
    "shares": "40000000.000000000000000000",
    "balance": {
      "denom": "stake",
      "amount": "40000000"
    }
  }
]

```
上述命令jack账户向kuchain16xlk6uf0ccsuh3zdff4lwpcafh7k6h6d6hzehk 投票40000000stake


#### 消息说明

```go
type MsgDelegate struct {
	DelegatorAddress types.AccountID 
	ValidatorAddress types.AccountID 
    Amount           types1.Coin     
}
```

如果出现以下情况，此消息将失败：

- 节点不存在
- 节点被惩罚
- 代币、面额和参数`params.BondDenom`不匹配

如果投票信息不存在，则将该信息对象创建做为该消息处理的一部分，否则，将更新现有的投票信息。

### 转投

转投就是将本来在A节点的票转到B节点

#### 命令说明

```bash
kucli tx kustaking redelegate [src-validator-addr] [dst-validator-addr] [amount] [flags]
```

+ src-validator-addr 转出票权的节点
+ dst-validator-addr 转入票权的节点
+ amount转投的金额
+ flags 我们这边只使用到一个From
    + from 为这个msg签名并支付手续费的账户，要保证from在转出票权的节点有足够的投票

 ```bash
./kucli tx kustaking redelegate kuchain16xlk6uf0ccsuh3zdff4lwpcafh7k6h6d6hzehk kuchain14f5d7f8u0e5xpgh0ty3ykflgyt7jepxlfchwnu 300stake --from jack

./kucli query kustaking delegations kuchain19674cxa9s6wl77scgj0nh445s3eqstwtw3g4nu
[
  {
    "delegator_address": "kuchain19674cxa9s6wl77scgj0nh445s3eqstwtw3g4nu",
    "validator_address": "kuchain14f5d7f8u0e5xpgh0ty3ykflgyt7jepxlfchwnu",
    "shares": "300.000000000000000000",
    "balance": {
      "denom": "stake",
      "amount": "300"
    }
  },
  {
    "delegator_address": "kuchain19674cxa9s6wl77scgj0nh445s3eqstwtw3g4nu",
    "validator_address": "kuchain16xlk6uf0ccsuh3zdff4lwpcafh7k6h6d6hzehk",
    "shares": "39999700.000000000000000000",
    "balance": {
      "denom": "stake",
      "amount": "39999700"
    }
  }
]
```

#### 消息说明

```go
type MsgBeginRedelegate struct {
	DelegatorAddress    types.AccountID 
	ValidatorSrcAddress types.AccountID 
	ValidatorDstAddress types.AccountID 
	Amount              types1.Coin     
}
```

如果出现以下情况，此消息将失败：

- 投票信息不存在
- 源节点或者目标节点不存在
- 转投的份额小于1个代币的份额
- 源节点收到还未生效的转投 (也就是说转投是不能传递的)
- 转投的份额超过了`params.MaxEntries`定义的最大值
- 代币的面额和`params.BondDenom`定义的面额不同

### 取消投票

取消投票就是将自己的投票从节点中取出，有一个21天的冻结期，冻结期过了以后才能到账

#### 命令说明

```bash
kucli tx kustaking unbond [validator-addr] [amount] [flags]
```

+ validator-addr 就是创建节点的时候输入的validator_name，可以在查询节点信息的时候查询到
+ amount 是投票的代币
+ flags 我们这边只使用到一个From
    + from 为这个msg签名并收取的账户

 ```bash
./kucli tx kustaking unbond  kuchain16xlk6uf0ccsuh3zdff4lwpcafh7k6h6d6hzehk 100stake --from jack

./kucli query kustaking unbonding-delegations kuchain19674cxa9s6wl77scgj0nh445s3eqstwtw3g4nu
[
  {
    "delegator_address": "kuchain19674cxa9s6wl77scgj0nh445s3eqstwtw3g4nu",
    "validator_address": "kuchain16xlk6uf0ccsuh3zdff4lwpcafh7k6h6d6hzehk",
    "entries": [
      {
        "creation_height": "1370",
        "completion_time": "2020-05-22T09:50:06.662743995Z",
        "initial_balance": "100",
        "balance": "100"
      }
    ]
  }
]
 ```

 #### 消息说明

 ```go
type MsgBeginUnbonding struct {
  DelegatorAddr sdk.AccAddress
  ValidatorAddr sdk.ValAddress
  Amount         sdk.Coin
}
```

如果出现以下情况，此消息将失败：

- 投票信息不存在
- 节点不存在
- 投票的份额少于要赎回的代币数额
- 现有的`UnbondingDelegation`已经超过`params.MaxEntries`定义的最大值了
- 代币面额和`params.BondDenom`定义的不同