# 主旨

本文档主要记录在kuchain开发的时候使用的相关命令

## 起链

```bash
./kucd init --chain-id=testing testing
./kucli keys add validator
./kucli keys add jack
./kucli keys add alice
./kucd add-genesis-account $(./kucli keys show validator -a) 1000000000stake,1000000000validatortoken
./kucd add-genesis-account $(./kucli keys show jack -a) 1000000000stake,1000000000validatortoken
./kucd add-genesis-account $(./kucli keys show alice -a) 1000000000stake,1000000000validatortoken
./kucd gentx --name validator
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

## 节点启动

```bash
./kucd start
# 这里可以等待一段时间，让第一个几点出块一段时间
./kucd start --home /home/xuyapeng/go_workspace/src/github.com/KuChain-io/data/kucddata1
```

## 在kustaking上面注册节点

```bash
./kucli tx kustaking create-validator \
  --amount=5stake \
  --pubkey=$(./kucd tendermint show-validator) \
  --moniker="jack test" \
  --chain-id=testing \
  --from=jack \
  --commission-rate="0.10"  
```

将第一个节点注册为节点

```bash
./kucli tx kustaking create-validator \
  --amount=5stake \
  --pubkey=$(./kucd tendermint show-validator --home /home/xuyapeng/go_workspace/src/github.com/KuChain-io/data/kucddata1) \
  --moniker="secode test" \
  --chain-id=testing \
  --from=alice \
  --commission-rate="0.10"
```

将第二个节点注册为节点

## 投票

```bash
./kucli tx kustaking delegate kuchainvaloper14g3r5zegtkvn4ls2v0sxpp006levx369e70udq 5000000stake --from jack
./kucli tx kustaking delegate kuchainvaloper1h79q0vwj4ffdmwelr0u9hhl794u556u5vtym5s 6000000stake --from alice
```

## 查看节点信息

```bash
 ./kucli query kustaking validators 
```

## 查看投票信息

```bash
./kucli query kustaking  delegations kuchain1h79q0vwj4ffdmwelr0u9hhl794u556u52xnynf
```

## 转投

```bash
./kucli tx kustaking redelegate kuchainvaloper1h79q0vwj4ffdmwelr0u9hhl794u556u5vtym5s kuchainvaloper14g3r5zegtkvn4ls2v0sxpp006levx369e70udq 300stake --from alice
```

转投的查询都为空，因为转投只需要1个区块就能生效

## 取消投票

```bash
./kucli tx kustaking unbond  kuchainvaloper14g3r5zegtkvn4ls2v0sxpp006levx369e70udq 100stake --from alice
```

## 查询取消的投票

```bash
./kucli query kustaking unbonding-delegations kuchain14g3r5zegtkvn4ls2v0sxpp006levx369lncr2e
```

## 查看用户代币信息

```bash
./kucli query bank balances kuchain1h79q0vwj4ffdmwelr0u9hhl794u556u52xnynf
```

