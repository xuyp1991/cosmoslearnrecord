# Future Improvements

当前文档仅描述了治理模块的最低可行产品。 未来的改进可能包括：

**`BountyProposals`:** 如果接受，`BountyProposal`将会创建一个公开的赏金，`BountyProposal`将会指定完成后会给出多少Atom,在`BountyProposal`被治理模块接收以后这些Atom将会从`reserve pool`领取，`reserve pool` 中的资金将会被锁定以便始终可以兑现付款，为了将`SoftwareUpgradeProposal`连接赏金，`SoftwareUpgradeProposal`将会使用一个`Proposal.LinkedProposal` 属性。如果连接公开赏金的`SoftwareUpgradeProposal`被治理组织通过的话，保留的资金将自动转给提交者

**Complex delegation:**  投票人可以选择节点以外的组织，最终代表链的将始终是节点，但投票人可以选择组织继承他们的投票权在节点获得他们投票权之前。换句话说当投票人选中的组织不投票的时候节点才能继承投票人的投票权

**Better process for proposal review:** `proposal.Deposit`的作用有2个部分，一方面防止垃圾提案，一方面奖励第三方审核员

