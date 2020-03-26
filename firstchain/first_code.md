# cosmos起链相关学习

本文记录第一次在cosmos框架上面手写代码,实现存储一个键值对功能

## 代码位置以及结构

代码结构如下

```tree
├── app
│   ├── app.go
│   └── export.go
├── cmd
│   ├── nscli
│   │   ├── main.go
│   │   └── nscli
│   └── nsd
│       ├── main.go
│       └── nsd
├── README.md
└── x
    └── easystore
        ├── client
        │   ├── cli
        │   │   ├── querier.go
        │   │   ├── storecmd.go
        │   │   └── trx.go
        │   └── rest
        │       └── rest.go
        ├── codec.go
        ├── genesis.go
        ├── handler.go
        ├── keeper.go
        ├── module.go
        ├── querier.go
        ├── README.md
        └── types
            ├── msg.go
            └── types.go
```

代码结构解读:
+ app模块       app模块是链的核心模块,用于连接各个子模块
+ cmd模块      cmd模块是命令模块,是封装相关命令的代码
+ x/easystore/client    这个模块是easystore模块的命令封装的模块
+ x/easystore/types     这个模块是easystore的一些参数和类型的定义,主要是用于外部可以直接调用到
+ x/easystore       这里面是easystore功能具体实现的部分

**之所以type模块要单独拆开是因为client/cli会调用types,easystore模块中的modules.go会调用client/cli模块,如果types不单独拆开会产生循环引用的情况**

## 代码编写记录

### app模块和cmd模块

第一个先说这两个模块,因为在初期者两个模块只需要复制gaia项目的相关go文件即可,不需要做任何改动.复制好以后直接编译运行,就可以启动一条cosmos的链了.

### easystore 模块

首先确定easystore模块的目的,因为是第一次编写,就想要一个最简单的功能,存储一个键值对.

### 相关结构和功能

```types.go
// Storedata 
type Storedata struct {
    Value string         `protobuf:"bytes,1,opt,name=value,proto3" json:"value"`
    Owner sdk.AccAddress `protobuf:"bytes,1,opt,name=value,proto3" json:"owner"`
}
```

定义一个结构体,里面有一个Value字段存储值.Owner是一个扩展字段,记录该键值对的归属,目前尚未用到

```msg.go
// MsgSetName defines a SetName message
type MsgSetStore struct {
    Name string
    Value  string
    Owner  sdk.AccAddress
}
```

定义存储键值对相关的msg.msg需要实现一些方法,以实现cosmos-sdk的types中Msg 接口.

```tx_msg.go
// Transactions messages must fulfill the Msg
type Msg interface {

    // Return the message type.
    // Must be alphanumeric or empty.
    Route() string

    // Returns a human-readable string for the message, intended for utilization
    // within tags
    Type() string

    // ValidateBasic does a simple validation check that
    // doesn't require access to any other information.
    ValidateBasic() Error

    // Get the canonical byte representation of the Msg.
    GetSignBytes() []byte

    // Signers returns the addrs of signers that must sign.
    // CONTRACT: All signatures must be present to be valid.
    // CONTRACT: Returns addrs in some deterministic order.
    GetSigners() []AccAddress
}
```

上面就是Msg interface的具体内容

一般情况下cosmos把具体实现的功能写在keeper里面.

```keeper.go
type Keeper struct {

    storeKey  sdk.StoreKey // Unexposed key to access store from sdk.Context

    cdc  *codec.Codec // The wire codec for binary encoding/decoding.
}

// Sets the entire Whois metadata struct for a name
func (k Keeper) SetStoredata(ctx sdk.Context, name string, storedata types.Storedata) {
    if storedata.Owner.Empty() {
        return
    }
    store := ctx.KVStore(k.storeKey)
    store.Set([]byte(name), k.cdc.MustMarshalBinaryBare(&storedata))
}

// SetOwner - sets the current owner of a name
func (k Keeper) Setvalue(ctx sdk.Context, name string, value string,owner sdk.AccAddress) {
    storedata := k.GetStoreData(ctx, name)
    storedata.Owner = owner
    storedata.Value = value
    k.SetStoredata(ctx, name, storedata)
}
```

这里只贴出来keeper的定义和最关键的两个函数,setValue 是需要被handler调用的

接下来就是handler的相关内容了

```handler.go
// // NewHandler returns a handler for "nameservice" type messages.
func NewHandler(k Keeper) sdk.Handler {
    return func(ctx sdk.Context, msg sdk.Msg) sdk.Result {
        ctx = ctx.WithEventManager(sdk.NewEventManager())
        switch msg := msg.(type) {
        case types.MsgSetStore:
            return handleMsgSetStore(ctx, k, msg)
        default:
        //    errMsg := fmt.Sprintf("Unrecognized nameservice Msg type: %v", msg.Type())
            return sdk.Result{}
        }
    }
}

// Handle a message to buy name
func handleMsgSetStore(ctx sdk.Context, keeper Keeper, msg types.MsgSetStore) sdk.Result {
    keeper.Setvalue(ctx,msg.Name,msg.Value,msg.Owner)
    return sdk.Result{}
}
```

NewHandler函数用于中转模块接收到的消息,handleMsgSetStore用于处理MsgSetStore消息.这里调用SetValue在链上存储一个键值对

以上就是setStore功能部分的具体内容了.

### 接口部分

想要将功能部分顺利接入到cosmos中,必须实现cosmos的相关接口,其中的核心部分在module.go里面,实现AppModuleBasic接口


```module.go
// AppModuleBasic is the standard form for basic non-dependant elements of an application module.
type AppModuleBasic interface {
    Name() string
    RegisterCodec(*codec.Codec)

    // genesis
    DefaultGenesis() json.RawMessage
    ValidateGenesis(json.RawMessage) error

    // client functionality
    RegisterRESTRoutes(context.CLIContext, *mux.Router)
    GetTxCmd(*codec.Codec) *cobra.Command
    GetQueryCmd(*codec.Codec) *cobra.Command
}

type AppModuleGenesis interface {
    AppModuleBasic
    InitGenesis(sdk.Context, json.RawMessage) []abci.ValidatorUpdate
    ExportGenesis(sdk.Context) json.RawMessage
}
```

AppModuleBasic 实现单独module需要的接口
AppModuleGenesis 实现module需要处理genesis文件需要的接口

```module.go
type AppModuleBasic struct{}

type AppModule struct {
    AppModuleBasic

    keeper        Keeper
}

// GetTxCmd returns the root tx command for the bank module.
func (AppModuleBasic) GetTxCmd(cdc *codec.Codec) *cobra.Command {
    return cli.GetTxCmd(cdc)
}

// GetQueryCmd returns no root query command for the bank module.
func (AppModuleBasic) GetQueryCmd(cdc *codec.Codec) *cobra.Command {
    //暂时先获取这个cmd
    return cli.GetQueryCmd(easystore.ModuleName,cdc)
}
```

这里定义对应的结构来实现接口.此处只列举2个函数GetTxCmd和GetQueryCmd,这两个函数返回的是client/cli中定义的相关命令

**这里是编译程序的时候出错最多的地方,写这个地方的时候需要特别注意.**

### 命令部分

命令部分主要是client/cli里面的内容

```tx.go
func GetTxCmd(cdc *codec.Codec) *cobra.Command {
    txCmd := &cobra.Command{
        Use:                        easystore.ModuleName,
        Short:                      "Auth transaction subcommands",
        DisableFlagParsing:         true,
        SuggestionsMinimumDistance: 2,
        RunE:                       client.ValidateCmd,
    }
    txCmd.AddCommand(
        GeteasystoreCmd(cdc),
        // GetSignCommand(cdc),
    )
    return txCmd
}
```

GetTxCmd只是定义了一个command,里面添加了GeteasystoreCmd返回的command

```tx.go
// GetBroadcastCommand returns the tx broadcast command.
func GeteasystoreCmd(cdc *codec.Codec) *cobra.Command {
    cmd := &cobra.Command{
        Use:   "store [name] [value]",
        Short: "store a value on chain",
        Long: strings.TrimSpace(`just store a value for a simple test
`),
        Args: cobra.ExactArgs(2),
        RunE: func(cmd *cobra.Command, args []string) (err error) {
        //    inBuf := bufio.NewReader(cmd.InOrStdin())
            cliCtx := context.NewCLIContext().WithCodec(cdc)
            txBldr := authtxb.NewTxBuilderFromCLI().WithTxEncoder(client.GetTxEncoder(cdc))

            // if cliCtx.Offline {
            //     return errors.New("cannot broadcast tx during offline mode")
            // }

            msgStoreData := types.NewMsgSetStore(args[0],args[1],cliCtx.GetFromAddress())
            return authclient.GenerateOrBroadcastMsgs(cliCtx, txBldr, []sdk.Msg{msgStoreData})
        },
    }
    return flags.PostCommands(cmd)[0]
}
```

GeteasystoreCmd的实现也比较简单,根据命令创建一个MsgSetStore,然后GenerateOrBroadcastMsgs签名并广播出去

最后rest部分不再详述.

### 将模块功能添加到app中

```app.go
    easystoretype "github.com/xuyp1991/cosaccount/x/easystore/types"
    easystore "github.com/xuyp1991/cosaccount/x/easystore"
```

首先在import上面加上我们使用的模块

```app.go
ModuleBasics = module.NewBasicManager(
    ......
        easystore.AppModuleBasic{},
    )
```

moduleBasics上面添加上AppModuleBasic

```app.go
type GaiaApp struct {
    ......
    easystoreKeeper   easystore.Keeper
        mm *module.Manager
}
    app.easystoreKeeper = easystore.NewKeeper(keys[gov.StoreKey],app.cdc)

```

把keerper添加到GaiaApp中,并在NewGaiaApp中对easystoreKeeper进行赋值

```app.go
app.mm = module.NewManager(
    ......
            easystore.NewAppModule(app.easystoreKeeper),
    )
```
将appModule添加上去

```app.go
    keys := sdk.NewKVStoreKeys(
        ......
        easystoretype.StoreKey,
    )
```

将storeKey添加上去以便创建对应的根键值对

```app.go
    app.mm.SetOrderInitGenesis(
        ......
        easystoretype.ModuleName,
    )
```

将ModuleName设置到可以初始化APP的列表中

这样easystore模块就添加到app里面了

### 在cmd模块添加对应的功能

在文件nscli/main.go里面

```main.go
import (
    ......
    easystorecmd "github.com/xuyp1991/cosaccount/x/easystore/client/cli"
    )

    ......

        txCmd.AddCommand(
            ......
                easystorecmd.GetTxCmd(cdc),
    )
    ......
```

添加对应引用以及将对应cmd添加到txCmd里面

### 运行并在链上存储相关键值对

```bash
./nsd init --chain-id=testing

./nscli keys add validator

./nsd add-genesis-account $(gaiacli keys show validator -a) 1000000000stake,1000000000validatortoken

./nsd gentx --nad collect-gentxsme validator

./nsd collect-gentxs

./nsd start
```

上述命令初始化一条链,chain-id是testing,创建一个账户validator并给一定初始代币,并将validator设置为验证者

```bash
./nscli config chain-id testing
./nscli config output json
./nscli config indent true
./nscli config trust-node true

./nscli tx easystore store something xrt --from validator
```

上述命令设置nscli的config,然后把键值对something xrt存放到easystore模块中

## query的编写

```types.go
    QueryValue   = "values"

type QueryResResolve struct {
    Value string `json:"value"`
}

func (this QueryResResolve) String() string {
    return fmt.Sprintf("value:%s", this.Value)
}
```

types.go上面添加查询命令的定义  values.查询返回的对应QueryResResolve以及对interface的实现函数String()

```keeper.go
func NewQuerier(keeper Keeper) sdk.Querier {
    return func(ctx sdk.Context, path []string, req abci.RequestQuery) ([]byte, sdk.Error) {
        switch path[0] {
        case types.QueryValue:
            return queryResolve(ctx, path[1:], req, keeper)

        default:
            return nil, sdk.ErrUnknownRequest("unknown bank query endpoint")
        }
    }
}
```

在NewQuerier函数中添加对types.QueryValue的处理

```querier.go
// nolint: unparam
func queryResolve(ctx sdk.Context, path []string, req abci.RequestQuery, keeper Keeper) (res []byte, err sdk.Error) {
    name := path[0]

    value := keeper.ResolveName(ctx, name)

    if value == "" {
        return []byte{}, sdk.ErrUnknownRequest("could not resolve name")
    }

    bz, err2 := codec.MarshalJSONIndent(keeper.cdc, QueryResResolve{value})
    if err2 != nil {
        panic("could not marshal result to JSON")
    }

    return bz, nil
}
```

新建文件querier.go添加对queryResolve的实现,调用ResolveName函数返回QueryResResolve对象

```tx.go
func GetQueryCmd(queryRoute string, cdc *codec.Codec) *cobra.Command {
    txCmd := &cobra.Command{
        Use:                        easystore.ModuleName,
        Short:                      "Auth transaction subcommands",
        DisableFlagParsing:         true,
        SuggestionsMinimumDistance: 2,
        RunE:                       client.ValidateCmd,
    }
    txCmd.AddCommand(
        GetCmdValue(queryRoute,cdc),
        // GetSignCommand(cdc),
    )
    return txCmd
}
```

tx.go上面添加GetQueryCmd函数,并将GetCmdValue添加到返回的command里面

```querier.go
func GetCmdValue(queryRoute string, cdc *codec.Codec) *cobra.Command {
    return &cobra.Command{
        Use:   "values [name]",
        Short: "query name store value",
        Args:  cobra.ExactArgs(1),
        RunE: func(cmd *cobra.Command, args []string) error {
            cliCtx := context.NewCLIContext().WithCodec(cdc)
            name := args[0]

            res, _,err := cliCtx.QueryWithData(fmt.Sprintf("custom/%s/values/%s", queryRoute, name), nil)
            if err != nil {
                fmt.Printf("could not resolve name - %s \n", string(name))
                return nil
            }

            var out types.QueryResResolve
            cdc.MustUnmarshalJSON(res, &out)
            return cliCtx.PrintOutput(out)
        },
    }
}
```

另建文件实现GetCmdValue函数

```moduler.go
// GetQueryCmd returns no root query command for the bank module.
func (AppModuleBasic) GetQueryCmd(cdc *codec.Codec) *cobra.Command {
    //暂时先获取这个cmd
    return cli.GetQueryCmd(easystore.ModuleName,cdc)
}
```

将cli.GetQueryCmd作为module的GetQueryCmd函数的返回

### 查询

``` bash
./nscli query easystore values something
{
  "value": "xrt"
}
```

至此,一个完整的模块编写完毕,虽然很简单,但作为第一次编写cosmos的模块中间也踩了很多坑.

## 版本说明

+ gaia               2.0.7
+ cosmos-sdk    0.37.8
+ tendminter        0.32.9

