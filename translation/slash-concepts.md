# Concepts

## States

在任意时间，链上都会注册任意数量的节点，每个块，最高的 `MaxValidators`（在`x/staking`中定义）没有被限制成为出块节点，也就是说他们可以进行出块并验证。出块节点是备受监督的，如果它们犯了协议错误，他们和投他们的用户的部门利益将会受损。

对于每一个节点我们都会保留一个`ValidatorSigningInfo`记录，保存验证器的活性以及其他违规的相关信息。

## Tombstone Caps

为了减轻最初可能会出现的非恶意的协议错误，Cosmos Hub为每一个节点设置了一个*tombstone* 上限，该限制仅允许当节点双签的时候对节点进行惩罚。例如：如果您对HSM进行错误配置，并对一堆旧块进行双重签名，则会因第一个双重签名而受到惩罚（然后立即标错），这仍然是相当昂贵的，并且需要避免，但*tombstone*会在一定成都上减少错误配置对经济的影响。

活动性故障没有上限，因为他们不能相互叠加，一旦发现违规就会立刻检测出活动性漏洞，节点会立刻被关小黑屋，所以一个节点不会犯下多个活动性错误。

## Infraction Timelines

为了说明`x/slashing`模块如何通过Tendermint共识来处理提交的证据，请考虑以下示例：

__Definitions__:

*[*   : timeline start  
*]*   : timeline end  
*C<sub>n</sub>* : infraction `n` committed  
*D<sub>n</sub>* : infraction `n` discovered  
*V<sub>b</sub>* : validator bonded  
*V<sub>u</sub>* : validator unbonded  

### Single Double Sign Infraction

<----------------->
[----------C<sub>1</sub>----D<sub>1</sub>,V<sub>u</sub>-----]

提交一次违规然后被发现，这个节点状态将会置为unbonded并为违规进行一次全额削减

### Multiple Double Sign Infractions

<--------------------------->
[----------C<sub>1</sub>--C<sub>2</sub>---C<sub>3</sub>---D<sub>1</sub>,D<sub>2</sub>,D<sub>3</sub>V<sub>u</sub>-----]

当多次违规行为提交被发现，此时，节点会因为第一次违规行为而被惩罚和关小黑屋，因为节点已经被*tombstone*，所以它不可能再成为出块节点

