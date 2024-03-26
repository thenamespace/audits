__Commit Hash__: `2f8c9d07dac4c849af66d3957f65cb871e815222` 
__Criterion__: [Immunefi Vulnerability Severity Classification System](https://immunefi.com/immunefi-vulnerability-severity-classification-system-v2-3/)

### Description
***The audit is done by Zhuo Zhang | izhuer.eth. PostDoc, Ph.D. in Computer Science @PurdueCS. Currently working at Offside Lab security company. This is the first audit of Namespace contracts done for the V1 version of the platform. Date of the Audit: July 21, 2023.***

### Revised
***Disclaimer: The namespace contract and the entire platform have grown and changed significantly since this audit was done.***

## Gas/Code Optimization

+ Function `ReservedRegistry.getVersions` is redundant, since `ReservedRegistry.versions` is _public_.
+ `ReservedRegistry.versions` / `WhiteLIstRegistry.versions` can be an integer rather rathe an array.
+  Change `i++` to `++i`.
+ It is not recommended to use `.length` in the loop header, we can put it out of the loop.
+ It is fine for `NameMintingController.withdraw` to be a public-callable function (save gas)
+ Put `onlyController` in low-level calls (e.g., `WhitelistRegistry._remove`) will cause more gas, especially for bulked functions.
+ Typo in `WhitelistREgistry.BullkRemovedFromWhitelist`, it should be `BulkRemovedFromWhitelist`
+ In `NameMintingRegistrar`, `NameWrapper` is currently fixed. Given the potential for future changes, it would be advisable to introduce a new function that allows for the update of `NameWrapper`.


## Low Issues

[L1] Lines 83-87 in `NameMintingRegistrar` seems incorrect: what if there is new white list type added. It is recommended to change it as `_validWhitelist`.

[L2] `NameMintingRegistrar.nodeAuthorization` should use `NameWrapper.canModifyName`, given the semantics of `NameWrapper.isApprovedForAll`.


## Medium Issues

### [M1] Intentional Reverts Caused by Malicious Sellers

__Type__: `Griefing`

A malicious seller, whom we'll refer to as Alice, has the potential to list her subnodes on the platform, employing specifically crafted configurations that preclude the possibility of any successful purchasing intentions. Through the exploitation of such a vulnerability, purchasers not only stand to lose their gas fees, but also risk undermining the platform's usability. For instance, an antagonist, who could possibly be a competitor, may list a number of subnodes, each with differing parent nodes. However, these subnodes cannot be legitimately sold, thus affecting the overall functionality and user experience of the platform.

To generate these _unsellable_ subnodes, Alice might employ the following tactics:
- Revoke the approval of `NameWrapperDelegate` after listing subnodes
- Transfer the parent nodes to another wallet (e.g., a different wallet controlled by Alice)
- Wait for the parent nodes to expire
- Burn the `CANNOT_CREATE_SUBDOMAIN` fuse of the parent node
- Generate a batch of desirable/reserved subnodes prior to their sale
- Do not burn the `CANNOT_UNWRAP` fuse of the parent node. This objective is slightly complicated to achieve as the `NameMintingRegistrar` enforces the burning of `CANNOT_UNWRAP` when listing subnodes. However, Alice could set a close expiry time for the parent node, list the subnodes, and upon expiration, Alice could reset the fuses of the parent node.

**Remedy Suggestion**: It is crucial that off-chain components continuously monitor these scenarios and issue alerts to users in case of any abnormal occurrences.

### [M2] Unbounded Gas Consumption Through Manipulation of `RegistryCalls._getSubdomainPrice`

__Type__: `Unbounded gas consumption`

In the `RegistryCalls._getSubdomainPrice` function, there is a loop that relies on input provided by sellers. The relevant code snippet is as follows:

```solidity
    for (uint256 i = 0; i < prices.length; i++) {
        if (prices[i].letters == labelLength) {
            return prices[i].price;
        }
    }
```

A malicious seller could potentially supply a list of irrelevant labelLengths (e.g., [1e10, 1e11, ...]), thereby forcing the buyer's transaction into a disproportionately large loop and causing an excessive gas expenditure.

__Remedy Suggestion__: To counter this, when adding or updating the prices, the contract should mandate that the input lengths are sorted. Additionally, in the `RegistryCalls` function, a binary search is recommended to improve efficiency and prevent this exploitative behaviour.


## High Issues

None

## Critical Issues

### Revised

_After careful examination, it's been established that the Subname front-running between Buyers and Node Owners is not a critical bug posing any threat to safety. It was designed like this by the Namespace team to improve user experience and cut down the registration time without having to wait for the commitment transaction in order to register a Subname. The commit and reveal mechanism mentioned below as a remedy for this issue was initially implemented by the Namespace team but removed after realizing it would significantly impact the user retention rate and overall usability of the platform with almost no security vulnerabilities for users losing funds or assets._

### [C1] Potential for Front-Running Between Buyers and Node Owners

No explicit protection is currently in place against front-running attacks between buyers and sellers, or among sellers themselves. This vulnerability is similar to the issue solved by the Ethereum Name Service (ENS) with a commit and reveal mechanism to protect name registrations.

Consider a scenario where a buyer, Alice, identifies a valuable name, such as `nick.ens`. Without a commit and reveal mechanism, a malicious user, Bob, could observe Alice's transaction, deem `nick.ens` valuable, and submit a new transaction to register `nick.ens` for himself, effectively front-running Alice's transaction.

The ENS addressed this problem through the use of a commit and reveal mechanism: [https://docs.ens.domains/contract-api-reference/.eth-permanent-registrar/controller](https://docs.ens.domains/contract-api-reference/.eth-permanent-registrar/controller)

A similar scenario could occur within NameSpace, where subnodes are treated as commodities and their uniqueness holds value to users. If Alice wants to buy `good.nick.ens`, the operation is susceptible to front-running attacks, e.g., Bob seizing the opportunity to buy good.nick.ens.

In summary, front-running between buyers and the node owners is possible, and several potential attack scenarios include:

- Attacks among buyers: A buyer could front-run another buyer's target, mirroring the scenario solved by ENS with the commit and reveal strategy.
- Attacks between buyers and sellers (the situation may exacerbate if a seller recently acquires the parent node from another user and intends to modify the lists.):
    - Update listing (buyer is the attacker): For instance, when a seller wishes to raise the price of their subnodes, a buyer could front-run the updateNameListing function, thereby securing the node at the former (lower) price.
    - Revoke whitelist (buyer is the attacker): This is akin to the approval attacks seen with ERC20 tokens. When a seller aims to remove a seller from the whitelist, that seller could front-run the `updateNameListing` transaction, purchasing subnodes that they should not have access to.
    - Update whitelist type (buyer is the attacker): In a similar vein, a buyer could exploit a seller's attempt to change the whitelist type.

The situation becomes worse when the seller just gets the parent node from someone else and wants to change the lists.

**Remedy Suggestion**: Implement a commit and reveal mechanism, akin to what ENS uses, to secure the transaction process and prevent front-running attacks.
