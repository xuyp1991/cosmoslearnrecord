# 本文档学习cosmos的提案相关的内容

提案相关的内容在gov模块.主动操作的功能有:submit-proposal,deposit,vote三个功能

## submit-proposal

submit-proposal是提交提案的功能,主要参数是

+ title 主体
+ description	描述
+ type 类型  text 文字描述,parameter_change 参数修改,software_upgrade 软件升级
+ deposit  存款,更准确来说是抵押

**在gov模块只提供text这一种类型的提案**

### 代码简析

handleMsgSubmitProposal 函数用于处理这个消息.这个函数主要调用SubmitProposal和AddDeposit来处理内容

+ SubmitProposal    将Proposal储存起来并更新proposalID
+ AddDeposit             对提案增加初始的押金

## deposit

deposit为提案增加押金,主要的参数是

+ proposal-id       提案ID
+ deposit               新增的抵押金额

### 代码简析

handleMsgDeposit函数用于处理这个消息,这个函数主要调用AddDeposit函数来添加对提案的押金

AddDeposit函数将用户的资金转移到模块里面,如果押金足够,调用activateVotingPeriod函数来启动投票

activateVotingPeriod函数修改对应proposal上面的参数(状态,启动投票时间等),将该proposal从InactiveProposalQueue移除,放置到ActiveProposalQueue中去

## vote

vote提供对提案进行投票的功能,主要的参数是

+ proposal-id       提案ID
+ option                投票的内容,有yes,no,no_with_veto,abstain 四种

### 代码简析

handleMsgVote 函数用于处理vote投票的消息,这个函数主要调用AddVote进行处理

addVote仅将投票相关信息进行记录

## 关于提案的处理

提案的处理是在EndBlock中进行处理,调用函数IterateActiveProposalsQueue进行处理.IterateActiveProposalsQueue遍历ActiveProposalsQueue中的所有超过endTime的提案,然后调用Tally进行处理

### Tally

Tally函数的功能是遍历该Proposal的所有vote,计算得出totalVotingPower,cosmos是根据票权来决定power的,也就是获得投票越多,Power越大.然后根据totalVotingPower和链上TotalBondedTokens相比较,获取proposal的状态.

+ 如果totalVotingPower不足TotalBondedTokens的Quorum,也就是没有足够的人投票,则提案失败,并没收押金
+ 如果投票人都投了弃权票,则投票失败,返还押金
+ 如果投票人中超过1/3投了veto,则提案失败,并没收押金
+ 如果投票人中除去弃权的票超过1/2投了通过,则提案通过,返还押金
+ 其他情况提案失败,并返还押金
  
Tally处理以后EndBlock根据Tally的处理修改Proposal的状态,执行没收或返还押金的操作,并将Proposal从ActiveProposalsQueue中移除.

