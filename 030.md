Stable Brick Goblin

medium

# Fillers' profit can be stolen by MEVbot or Block Proposer

## Summary
Orders are executed by fillers, who are expected to run an off-chain logic, and pick orders, then get them to  execute for profit. The issue is MEVbot or Block Proposers can front run the execution and just replace the filler role to get profit. They have not to do any off-chain work, just monitor the node mempool and extract risk free profit from fillers.

## Vulnerability Detail
In current implementation, the ````exclusive/filler```` field is optional (L65), any address could be a filler.
```solidity
File: gladius-contracts-internal/src/lib/ExclusivityOverrideLib.sol
60:     function hasFillingRights(
61:         address exclusive,
62:         uint256 exclusivityEndTime
63:     ) internal view returns (bool pass) {
64:         return
65:             exclusive == address(0) ||
...
68:     }

```  
And the ````orders```` and ````quantities```` parameters could be fetched from pending ````executeBatch```` transaction of mempool. Then MEVbot or Block Proposers can front run the execution to get profit. The original transaction would fail and suffer both order profit and gas loss.
```solidity
File: gladius-contracts-internal/src/reactors/BaseGladiusReactor.sol
082:     function executeBatch(
083:         SignedOrder[] calldata orders,
084:         uint256[] calldata quantities
085:     ) external payable override nonReentrant {
...
101:     }

```


## Impact
Fillers' profit can be stolen by MEVbot or Block Proposer.

## Code Snippet
https://github.com/sherlock-audit/2024-02-rubicon-finance/blob/11cac67919e8a1303b3a3177291b88c0c70bf03b/gladius-contracts-internal/src/lib/ExclusivityOverrideLib.sol#L65

## Tool used

Manual Review

## Recommendation
Haven't see a perfect solution, maybe adding a filler whitelist
