# 浅析cosmos的capability模块

## 背景

在ICS 5标准中提到了模块是区块链上相互独立且互不信任的单元，区块链间通信的本质上是模块之间的通信。为了实现模块之间的通信，模块需要绑定一定数量的端口，并连接到另一个模块绑定的端口，两个端口的连接形成一个通道(一个端口可以绑定到多个通道上面)，使模块之间可以进行通信。模块可以把绑定的端口暴露给一个特定权限的模块(状态机)，状态机也可以自定义规则来约束模块可以绑定的端口。状态机如果要实现对模块及其端口的管理，必须有校验模块的capability key的功能。capability key是当端口和通道初始化时创建并在验证端口和通道时使用的，而且状态机必须有创建capability key的功能，状态机还需要提供保存capability key以及在重启或启动时重新生成capability key的功能。

capability模块为状态机提供关于capability key的创建，运行时配置，校验和跟踪的功能。在app.go中的应用程序初始化期间，CapabilityKeeper会通过唯一的函数引用(ScopeToModule)连接模块，以便在后续的调用中识别出该模块。在磁盘加载初始状态时，CapabilityKeeper的Initialise功能会为先前分配的capability创建新的capability key，并将其保存在内存中。CapabilityKeeper包含一个持久化的储存，一个MemoryStore和内存映射。MemoryStore储存从模块和capability key到capability name的正向映射和一个从模块和capability name到capability key的反向映射。

## 功能列举

CapabilityKeeper实现了以下的定义和功能

```go
type Capability struct {
  index uint64
}
```

Capability是一种结构，它储存的地址指向实际的功能地址。

```go
    Keeper struct {
        cdc           codec.Marshaler
        storeKey      sdk.StoreKey
        memKey        sdk.StoreKey
        capMap        map[uint64]*types.Capability
        scopedModules map[string]struct{}
        sealed        bool
    }
        ScopedKeeper struct {
        cdc      codec.Marshaler
        storeKey sdk.StoreKey
        memKey   sdk.StoreKey
        capMap   map[uint64]*types.Capability
        module   string
    }
```

keeper提供了创建ScopedKeeper并将其和模块名称绑定的功能。ScopedKeeper必须在应用程序初始化时创建，并传递给模块，然后状态机可以使用capability name来绑定Capability或者创建新的Capability以便状态机识别和跟踪模块传递的Capability

```go
func (k *Keeper) ScopeToModule(moduleName string) ScopedKeeper {
    if k.sealed {
        panic("cannot scope to module via a sealed capability keeper")
    }

    if _, ok := k.scopedModules[moduleName]; ok {
        panic(fmt.Sprintf("cannot create multiple scoped keepers for the same module name: %s", moduleName))
    }

    k.scopedModules[moduleName] = struct{}{}

    return ScopedKeeper{
        cdc:      k.cdc,
        storeKey: k.storeKey,
        memKey:   k.memKey,
        capMap:   k.capMap,
        module:   moduleName,
    }
}
```

ScopeToModule用于创建具有特定名称(模块名)的子作用域，模块名必须保持唯一。该函数必须在InitializeAndSeal之前进行调用。

```go
func (k *Keeper) InitializeAndSeal(ctx sdk.Context) {
    if k.sealed {
        panic("cannot initialize and seal an already sealed capability keeper")
    }
    memStore := ctx.KVStore(k.memKey)
    memStoreType := memStore.GetStoreType()
    if memStoreType != sdk.StoreTypeMemory {
        panic(fmt.Sprintf("invalid memory store type; got %s, expected: %s", memStoreType, sdk.StoreTypeMemory))
    }
    prefixStore := prefix.NewStore(ctx.KVStore(k.storeKey), types.KeyPrefixIndexCapability)
    iterator := sdk.KVStorePrefixIterator(prefixStore, nil)
    // initialize the in-memory store for all persisted capabilities
    defer iterator.Close()
    for ; iterator.Valid(); iterator.Next() {
        index := types.IndexFromKey(iterator.Key())
        var capOwners types.CapabilityOwners
        k.cdc.MustUnmarshalBinaryBare(iterator.Value(), &capOwners)
        k.InitializeCapability(ctx, index, capOwners)
    }
    k.sealed = true
}
```

在加载初始状态并创建所有必要的scopedModules之后，必须仅对一次InitializeAndSeal进行一次调用，以便根据特定模块先前声明的Capability用新创建的Capability填充内存存储，并防止创建任何新的ScopedKeeper。

```go
func (sk ScopedKeeper) NewCapability(ctx sdk.Context, name string) (*types.Capability, error) {
    store := ctx.KVStore(sk.storeKey)
    if _, ok := sk.GetCapability(ctx, name); ok {
        return nil, sdkerrors.Wrapf(types.ErrCapabilityTaken, fmt.Sprintf("module: %s, name: %s", sk.module, name))
    }
    // create new capability with the current global index
    index := types.IndexFromKey(store.Get(types.KeyIndex))
    cap := types.NewCapability(index)
    if err := sk.addOwner(ctx, cap, name); err != nil {
        return nil, err
    }
    store.Set(types.KeyIndex, types.IndexToKey(index+1))
    memStore := ctx.KVStore(sk.memKey)
    memStore.Set(types.FwdCapabilityKey(sk.module, cap), []byte(name))
    memStore.Set(types.RevCapabilityKey(sk.module, name), sdk.Uint64ToBigEndian(index))
    sk.capMap[index] = cap
    logger(ctx).Info("created new capability", "module", sk.module, "name", name)
    return cap, nil
}
```

任何模块都可以通过调用NewCapability来创建唯一的，不可伪造的Capability，新创建的Capability会自动保存，并和Capability name进行绑定，调用者不必再次调用ClaimCapability功能。

```go
func (sk ScopedKeeper) AuthenticateCapability(ctx sdk.Context, cap *types.Capability, name string) bool {
    return sk.GetCapabilityName(ctx, cap) == name
}
```

任何模块都可以通过调用AuthenticateCapability来校验Capability和Capability name是否匹配。

```go
func (sk ScopedKeeper) ClaimCapability(ctx sdk.Context, cap *types.Capability, name string) error {
    if err := sk.addOwner(ctx, cap, name); err != nil {
        return err
    }
    memStore := ctx.KVStore(sk.memKey)
    memStore.Set(types.FwdCapabilityKey(sk.module, cap), []byte(name))
    memStore.Set(types.RevCapabilityKey(sk.module, name), sdk.Uint64ToBigEndian(cap.GetIndex()))
    logger(ctx).Info("claimed capability", "module", sk.module, "name", name, "capability", cap.GetIndex())
    return nil
}
```

ClaimCapability允许一个模块声明从另外一个模块接收Capability，以便将来GetCapability调用成功，如果模块希望可以通过名称来访问一个Capability，则必须调用ClaimCapability，一个Capability的Owner是可以有很多个，如果几个模块有相同的Capability，它们都是这个Capability的Owner.

```go
func (sk ScopedKeeper) GetCapability(ctx sdk.Context, name string) (*types.Capability, bool) {
    memStore := ctx.KVStore(sk.memKey)
    key := types.RevCapabilityKey(sk.module, name)
    indexBytes := memStore.Get(key)
    index := sdk.BigEndianToUint64(indexBytes)
    if len(indexBytes) == 0 {
        delete(sk.capMap, index)
        return nil, false
    }
    cap := sk.capMap[index]
    if cap == nil {
        memStore.Delete(key)
        return nil, false
    }
    return cap, true
}
```

GetCapability允许模块根据Capability name获取其所拥有的Capability。 不允许模块检索其不拥有的Capability。

```go
func (sk ScopedKeeper) ReleaseCapability(ctx sdk.Context, cap *types.Capability) error {
    name := sk.GetCapabilityName(ctx, cap)
    if len(name) == 0 {
        return sdkerrors.Wrap(types.ErrCapabilityNotOwned, sk.module)
    }
    memStore := ctx.KVStore(sk.memKey)
    memStore.Delete(types.FwdCapabilityKey(sk.module, cap))
    memStore.Delete(types.RevCapabilityKey(sk.module, name))
    capOwners := sk.getOwners(ctx, cap)
    capOwners.Remove(types.NewOwner(sk.module, name))
    prefixStore := prefix.NewStore(ctx.KVStore(sk.storeKey), types.KeyPrefixIndexCapability)
    indexKey := types.IndexToKey(cap.GetIndex())
    if len(capOwners.Owners) == 0 {
        prefixStore.Delete(indexKey)
        delete(sk.capMap, cap.GetIndex())
    } else {
        prefixStore.Set(indexKey, sk.cdc.MustMarshalBinaryBare(capOwners))
    }
    return nil
}
```

ReleaseCapability运行模块根据Capability name释放其拥有的Capability，如果Capability没有其他Owner,Capability将会被删除。

## 小结

capability模块为状态机实现对模块功能的跟踪和校验，以便状态机可以实现接收不同模块的消息以实现跨链通信的功能。