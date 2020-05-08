# kuchain命令总结
本文档总结部分kuchain的命令，并对命令进行说明

## 命令条目

+ init
+ gentx
+ collect-gentxs
+ start
+ tx kustaking 
  + create-validator
  +  edit-validator
  +  delegate
  +  redelegate
  +  unbond 
+ query kustaking 
  + validators
  + validator
  + delegations 
  + unbonding
+ tx kuslashing unjail
+  tx kugov 
   +  submit-proposal
   +  deposit
   +  vote
+  query kugov
   +  proposals
   +  proposal
   +  votes
+  tx kugov submit-proposal param-change

## init

init命令用于初始化一个节点

```bash
./kucd init --chain-id=testing secondtest  --home /home/xuyapeng/go_workspace/src/github.com/KuChain-io/data/kucddata1
```

+ --chain-id 用于指明该节点连接的链的chain-id
+ secondtest是该节点的monitor  本地存储的一个名字
+ --home    用于指定节点文件所在的目录，默认是~/.kucd

## gentx

该命令生成一个注册节点和给节点投票的msg,用于起链的时候创建创世节点

```bash
./kucd gentx kuchain14f5d7f8u0e5xpgh0ty3ykflgyt7jepxlfchwnu  --name validator
```

+ kuchain14f5d7f8u0e5xpgh0ty3ykflgyt7jepxlfchwnu 是节点的名字，目前最好使用用户的地址，后续会修改为account的name
+ --name validator 使用钱包中哪个账户来对msg进行签名

```json
{
	"type": "cosmos-sdk/StdTx",
	"value": {
		"msg": [{
			"type": "kuchain/MsgCreateValidator",
			"value": {
				"description": {
					"moniker": "testing"
				},
				"CommissionRates": "0.100000000000000000",
				"validator_address": "kuchain14f5d7f8u0e5xpgh0ty3ykflgyt7jepxlfchwnu",
				"delegator_address": "kuchain14f5d7f8u0e5xpgh0ty3ykflgyt7jepxlfchwnu",
				"pubkey": "kuchainvalconspub1zcjduepq5u5f4n38gnk4d6465s9vntjvxfrwqrlma4xjpwx06zd4hrfayrfqtqegu8"
			}
		}, {
			"type": "kuchain/MsgDelegate",
			"value": {
				"delegator_address": "kuchain14f5d7f8u0e5xpgh0ty3ykflgyt7jepxlfchwnu",
				"validator_address": "kuchain14f5d7f8u0e5xpgh0ty3ykflgyt7jepxlfchwnu",
				"amount": {
					"denom": "stake",
					"amount": "100000000"
				}
			}
		}],
		"fee": {
			"amount": [],
			"gas": "200000"
		},
		"signatures": [{
			"pub_key": {
				"type": "tendermint/PubKeySecp256k1",
				"value": "A7H83IkGSOTc+RwKsD4O/eKf6TFq2DWUBHktcn8TZSOF"
			},
			"signature": "ja7wHrquHV6ecLtmWQr36Zma8BQlpi4kyA/13qYTMfFZeRkRAMb/z/XKUQu+owTDVaxyK2oYXAg0JHJWha6qLw=="
		}],
		"memo": "fe7e3031d0b4a590091c3aa431541755bb644931@192.168.1.170:26656"
	}
}
```

上面就是通过命令生成的msg,msg上面有2个操作，一个是`kuchain/MsgCreateValidator` 注册节点，一个是`kuchain/MsgDelegate` 给节点投票。

## collect-gentxs

该命令是将gentxs生成的msg写入到genesis.json文件中

##  start

启动命令，可以添加`--home`指定从哪个文件夹中启动

## tx kustaking

### create-validator

注册节点命令。

```bash
./kucli tx kustaking create-validator kuchain16xlk6uf0ccsuh3zdff4lwpcafh7k6h6d6hzehk\
  --pubkey=$(./kucd tendermint show-validator --home /home/xuyapeng/go_workspace/src/github.com/KuChain-io/data/kucddata1) \
  --moniker="secode test" \
  --chain-id=testing \
  --from=alice \
  --commission-rate="0.10"
```

+ kuchain16xlk6uf0ccsuh3zdff4lwpcafh7k6h6d6hzehk  是节点的名字，目前最好使用用户的地址，后续会修改为account的name
+ pubkey  节点的公钥，使用`./kucd tendermint show-validator`获取
+ moniker 节点在本地的一个名字，会在链上的description中存储
+ chain-id 链的id
+ from  支付手续费和进行签名的用户名(钱包中存储的用户名)
+ commission-rate  佣金比例

### edit-validator

修改节点信息，此处举例修改佣金比例

```bash
./kucli tx kustaking  edit-validator  kuchain16xlk6uf0ccsuh3zdff4lwpcafh7k6h6d6hzehk   --commission-rate="0.20"  --from=alice
```

+ kuchain16xlk6uf0ccsuh3zdff4lwpcafh7k6h6d6hzehk 是需要修改的节点的名字
+ commission-rate 指要修改的佣金比例
+ from 对msg进行签名和付手续费的账户名字

**注意：24小时内只能修改1次**

### delegate

投票给节点

```bash
./kucli tx kustaking delegate kuchain16xlk6uf0ccsuh3zdff4lwpcafh7k6h6d6hzehk 5600stake --from alice
```

+ kuchain16xlk6uf0ccsuh3zdff4lwpcafh7k6h6d6hzehk 是接收投票的节点的名字
+ 5600stake投票的金额
+ from 投票的账户

**上述命令实现的功能是alice账户将5600stake投票给kuchain16xlk6uf0ccsuh3zdff4lwpcafh7k6h6d6hzehk节点**

### redelegate

转投，将投票从一个节点转到另外一个节点

```bash
./kucli tx kustaking redelegate kuchain16xlk6uf0ccsuh3zdff4lwpcafh7k6h6d6hzehk jack 300stake --from alice
```

+ kuchain16xlk6uf0ccsuh3zdff4lwpcafh7k6h6d6hzehk 转出节点的名字
+ jack 转入节点的名字
+ 300stake 转投的金额
+ alice 转投的账户

**上述命令实现的功能是alice账户将300stake从kuchain16xlk6uf0ccsuh3zdff4lwpcafh7k6h6d6hzehk节点转到jack节点**

### unbond

取消投票

```bash
./kucli tx kustaking unbond kuchain16xlk6uf0ccsuh3zdff4lwpcafh7k6h6d6hzehk 100stake --from alice
```

+ kuchain16xlk6uf0ccsuh3zdff4lwpcafh7k6h6d6hzehk 取消投票的节点
+ 100stake 取消节点的币量
+ alice 取消投票的账户
  
  **上述命令实现的功能是alice账户从kuchain16xlk6uf0ccsuh3zdff4lwpcafh7k6h6d6hzehk节点取消100stake的投票**

## query kustaking 

### validators

查询所有的节点

```bash
 ./kucli query kustaking validators
```

### validator

查询单个节点信息

```bash
 ./kucli query kustaking validator kuchain14f5d7f8u0e5xpgh0ty3ykflgyt7jepxlfchwnu
```

+ kuchain14f5d7f8u0e5xpgh0ty3ykflgyt7jepxlfchwnu是注册节点的时候传入的节点的名字

### delegations

查询单个用户全部投票信息

```bash
./kucli query kustaking  delegations kuchain14f5d7f8u0e5xpgh0ty3ykflgyt7jepxlfchwnu
```
  
+ kuchain14f5d7f8u0e5xpgh0ty3ykflgyt7jepxlfchwnu是投票的账户的地址 

### delegation

查询单个用户在某一个节点上面的投票信息

```bash
./kucli query kustaking  delegation kuchain14f5d7f8u0e5xpgh0ty3ykflgyt7jepxlfchwnu jack
```

+ kuchain14f5d7f8u0e5xpgh0ty3ykflgyt7jepxlfchwnu  第一个是投票的用户的地址
+ jack 第二个是节点的名称
  
**上述命令实现了查询kuchain14f5d7f8u0e5xpgh0ty3ykflgyt7jepxlfchwnu在jack上面的投票信息**

### unbonding

查询取消投票的内容

```bash
./kucli query kustaking unbonding-delegations kuchain14f5d7f8u0e5xpgh0ty3ykflgyt7jepxlfchwnu
```

+ kuchain14f5d7f8u0e5xpgh0ty3ykflgyt7jepxlfchwnu 是取消投票的用户的地址

**上述功能实现了查询kuchain14f5d7f8u0e5xpgh0ty3ykflgyt7jepxlfchwnu所有取消投票的信息**

## unjail

解除惩罚命令，用于节点因为漏块或者双签导致被惩罚，过惩罚期以后使用该命令使节点恢复正常

```bash
./kucli tx kuslashing unjail --from validator
```

+ validator 注册节点时使用的名字所对应的在钱包中的账户，也就是说validator账户的地址就是节点的名字

## tx kugov 

### submit-proposal

提交提案，此处仅说明一下提交一个文本提案

```bash
./kucli tx kugov submit-proposal --title "jack for test" --description "test test test" --deposit 100000000stake --type text  --from alice
```
+  title  提案主体
+  description 提案说明
+  deposit 提案押金
+  type 提案类型，此处只有text
+  from 给msg签名并支付费用和押金的账户

### deposit

给提案添加押金

```bash
./kucli tx kugov deposit 1 500000000stake --from jack
```

+ 1 是提案的ID，增加押金的时候需要指明给哪个提案增加押金
+ 500000000stake 是增加的押金的数目
+ from 是支付押金的用户

### vote

对提案进行投票

```bash
./kucli tx kugov vote 1 yes --from validator
```

+ 1 是提案的ID
+ yes 是投票的内容，有yes/no/no_with_veto/abstain 四种
+ from 进行投票的节点的账户
  
## query kugov

### proposals

查询所有提案的内容

```bash
./kucli query kugov proposals
```

### proposal

查询单个提案的信息

```bash
./kucli query kugov proposal 1
```

+ 1 是提案的ID

### votes

查询提案的投票内容

```bash
./kucli query kugov votes 1
```

+ 1 是提案的ID

##  tx kugov submit-proposal param-change

提案修改参数

```bash
./kucli tx kugov submit-proposal param-change   "./proposal.json" --from jack
```

**上诉命令实现了根据proposal.json的内容来修改某一个模块的参数的一个提案**