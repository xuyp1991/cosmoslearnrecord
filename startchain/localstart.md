# gaia启动

## 本地单节点链

```bash
# 初始化genesis.json文件，它将帮助您引导网络
gaiad init --chain-id=testing testing

# 创建一个密钥来保存您的验证者帐户
gaiacli keys add validator

# 将该密钥添加到genesis文件中的genesis.app_state.accounts数组中
#注意：此命令使您可以设置硬币数。 确保此帐户有一些带有genesis.app_state.staking.params.bond_denom代币的硬币，默认为staking
gaiad add-genesis-account $(gaiacli keys show validator -a) 1000000000stake,1000000000validatortoken

# 生成创建验证器的交易
gaiad gentx --name validator

# 将生成的绑定交易添加到创世文件
gaiad collect-gentxs

# 现在可以安全地启动" gaiad"了
gaiad start
```

## 加入测试链

```bash
# 初始化节点并创建必要的配置文件
gaiad init <your_custom_moniker>

#下载13006的genesis.json文件
curl https://raw.githubusercontent.com/cosmos/testnets/master/gaia-13k/13006/genesis.json > ./genesis.json


#下载最新的config配置
curl https://raw.githubusercontent.com/cosmos/testnets/master/latest/config.toml > $HOME/.gaiad/config/config.toml

#删除过时的数据
gaiad unsafe-reset-all

gaiad start
```
