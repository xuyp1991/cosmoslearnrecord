# 主旨

本文档主要记录在kuchain开发的时候使用的相关命令

## 起链

```bash
./kucli keys add validator
./kucli keys add jack
./kucli keys add alice




./kucd init --chain-id=testing testing
./kucd add-genesis-account $(./kucli keys show validator -a) 10000000000000000stake,1000000000validatortoken
./kucd add-genesis-account $(./kucli keys show jack -a) 10000000000000000stake,1000000000validatortoken
./kucd add-genesis-account $(./kucli keys show alice -a) 10000000000000000stake,1000000000validatortoken
./kucd gentx kuchain14f5d7f8u0e5xpgh0ty3ykflgyt7jepxlfchwnu  --name validator
./kucd collect-gentxs

./kucli config chain-id testing
./kucli config output json
./kucli config indent true
./kucli config trust-node true
```

上面是第一个节点的准备起链的命令，可以使用`./kucd start`直接启动，这边我启动后停了一下，因为要把genesis.json文件拿出来

## 第二个节点

```bash
./kucd init --chain-id=testing secondtest  --home /home/xuyapeng/go_workspace/src/github.com/KuChain-io/data/kucddata1
# 复制genesis.json文件
# 修改config.toml文件，将默认的端口都使用3开头的5位端口
# 获取到第一个节点的ID
./kucd tendermint show-node-id
# 根据第一个节点的ID拼接出persistent_peers   该字段是在config.toml的[p2p]模块下
persistent_peers="7ff250764dde19bfccf4c6301a22cae2dba5d241@127.0.0.1:26656"
```

## 第三个节点

```bash
./kucd init --chain-id=testing thirdtest  --home /home/xuyapeng/go_workspace/src/github.com/KuChain-io/data/kucddata2

```

## 第四个节点 

```bash
./kucd init --chain-id=testing doubletest --home /home/xuyapeng/go_workspace/src/github.com/KuChain-io/data/kucddata3

```

## 节点启动

```bash
./kucd start
# 这里可以等待一段时间，让第一个几点出块一段时间
./kucd start --home /home/xuyapeng/go_workspace/src/github.com/KuChain-io/data/kucddata1
```

## 在kustaking上面注册节点

```bash
./kucli tx kustaking create-validator jack \
  --amount=5stake \
  --pubkey=$(./kucd tendermint show-validator --home /home/xuyapeng/go_workspace/src/github.com/KuChain-io/data/kucddata2) \
  --moniker="jack test" \
  --chain-id=testing \
  --from=jack \
  --commission-rate="0.10"  
```

将第一个节点注册为节点

```bash
./kucli tx kustaking create-validator kuchain16xlk6uf0ccsuh3zdff4lwpcafh7k6h6d6hzehk\
  --amount=5stake \
  --pubkey=$(./kucd tendermint show-validator --home /home/xuyapeng/go_workspace/src/github.com/KuChain-io/data/kucddata1) \
  --moniker="secode test" \
  --chain-id=testing \
  --from=alice \
  --commission-rate="0.10"
```

将第二个节点注册为节点

## 修改节点信息

```bash
./kucli tx kustaking  edit-validator  kuchain16xlk6uf0ccsuh3zdff4lwpcafh7k6h6d6hzehk   --commission-rate="0.20"  --from=alice
```


## 投票

```bash
./kucli tx kustaking delegate validator 5500stake --from jack
./kucli tx kustaking delegate kuchain16xlk6uf0ccsuh3zdff4lwpcafh7k6h6d6hzehk 5600stake --from alice
./kucli tx kustaking delegate kuchain16xlk6uf0ccsuh3zdff4lwpcafh7k6h6d6hzehk 40000000stake --from jack

kuchain16xlk6uf0ccsuh3zdff4lwpcafh7k6hccrez08

./kucli tx kustaking delegate kuchainvaloper16xlk6uf0ccsuh3zdff4lwpcafh7k6h6du64xs0 200000000stake --from jack
./kucli tx kustaking delegate kuchainvaloper16xlk6uf0ccsuh3zdff4lwpcafh7k6h6du64xs0 156000000stake --from alice

./kucli tx kustaking delegate kuchainvaloper1hgc32us5222xq35nfq3zcm8ysz6qj0cfsry0ut 100000000stake --from jack
./kucli tx kustaking delegate kuchainvaloper1hgc32us5222xq35nfq3zcm8ysz6qj0cfsry0ut 156000000stake --from alice

```

## 查看节点信息

```bash
 ./kucli query kustaking validators
 ./kucli query kuslashing signing-info kuchainvalconspub1zcjduepq3hcjlqj38nrzn7usfqgzay3v0793477kvw9zszgjnmqq0ml7847smdrrs7
```

## 查看投票信息

```bash
./kucli query kustaking  delegations kuchain1hgc32us5222xq35nfq3zcm8ysz6qj0cfkwnsmj
./kucli query kustaking  delegations kuchain16xlk6uf0ccsuh3zdff4lwpcafh7k6h6d6hzehk
```

## 转投

```bash
./kucli tx kustaking redelegate jack validator 300stake --from alice
./kucli tx kustaking redelegate kuchainvaloper19674cxa9s6wl77scgj0nh445s3eqstwtgul259 kuchainvaloper16xlk6uf0ccsuh3zdff4lwpcafh7k6h6du64xs0 300stake --from alice
```

转投的查询都为空，因为转投只需要1个区块就能生效

## 取消投票

```bash
./kucli tx kustaking unbond  validator 100stake --from alice
```

## 查询取消的投票

```bash
./kucli query kustaking unbonding-delegations kuchain16xlk6uf0ccsuh3zdff4lwpcafh7k6h6d6hzehk
```

## 查看用户代币信息

```bash
./kucli query bank balances kuchain16xlk6uf0ccsuh3zdff4lwpcafh7k6h6d6hzehk
```

## 因为漏块而关小黑屋的解救命令

```bash
 ./kucli tx kuslashing unjail --from alice
```

## 在kuchain 上进行提案

```bash
./kucli tx kugov submit-proposal --title "jack for test" --description "test test test" --deposit 100000000stake --type text  --from alice
./kucli tx kugov deposit 3 500000000stake --from jack
```

##  查看所有提案

```bash
./kucli query kugov proposals
```

##  同意提案

```bash
./kucli tx kugov vote 3 yes --from validator
./kucli tx kugov vote 5 abstain --from alice
./kucli tx kugov vote 2 yes --from jack


./kucli tx kugov vote 2 yes --from validator
./kucli tx kugov vote 3 yes --from validator

./kucli tx kugov vote 6 yes --from alice
./kucli tx kugov vote 6 yes --from validator
```

## 查看投票相关信息

```bash
./kucli query kugov votes 1
```

## 修改参数的提案

```bash
./kucli tx kugov submit-proposal param-change   "./proposal.json" --from jack

./kucli tx kugov submit-proposal param-change   "./slashproposal.json" --from jack

./kucli tx kugov submit-proposal param-change   "./proposalevidence.json" --from jack

./kucli tx kugov submit-proposal param-change   "./mintproposal.json" --from jack

./kucli tx kugov submit-proposal param-change   "./govproposal.json" --from jack
```











## 在cosmos的staking上面注册节点

```bash
./kucli tx staking create-validator \
  --amount=5stake \
  --pubkey=$(./kucd tendermint show-validator --home /home/xuyapeng/go_workspace/src/github.com/KuChain-io/data/kucddata1) \
  --moniker="jack test" \
  --chain-id=testing \
  --from=jack \
  --commission-rate="0.10"   \
  --commission-max-rate="0.20" \
  --commission-max-change-rate="0.01"  \
  --min-self-delegation=1
  ```

## 在cosmos的staking上面投票

  ```bash
./kucli tx staking delegate kuchainvaloper1qx7kdeddn60rqt3tvv592293tegz34sy5u2j9s 55000000stake --from jack
./kucli tx staking delegate kuchainvaloper1xsdg0nu9czx89p54xtmc589uds98j8kdhsvl0l 56000000stake --from alice
  ```


```bash
kuchain1xsdg0nu9czx89p54xtmc589uds98j8kd3amqgx
kuchain1puddfw5kyeq57n0mel66yk0hf8h7ryrpntvs72
```
