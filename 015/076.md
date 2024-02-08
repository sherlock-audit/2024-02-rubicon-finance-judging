Brave Teal Starling

high

# Fees is not injected properly

## Summary
Instead of injecting fees on the correct resolved order the function injects the fees on its copy
## Vulnerability Detail
Following is execute functions (it applies to all other execute types)
```solidity
function execute(
        SignedOrder calldata order,
        uint256 quantity
    ) external payable override nonReentrant {
        ResolvedOrder[] memory resolvedOrders = new ResolvedOrder[](1);
        resolvedOrders[0] = resolve(order, quantity);

        _prepare(resolvedOrders);
        _fill(resolvedOrders);
    }
```
_prepare function is meant to update resolvedOrders but it updates and injects fees on the copy of resolvedOrders as can be seen from the following highlighted lines
```solidity
function _prepare(ResolvedOrder[] memory orders) internal {
        uint256 ordersLength = orders.length;
        unchecked {
            for (uint256 i = 0; i < ordersLength; i++) {
    @==>            ResolvedOrder memory order = orders[i];
    @==>         _injectFees(order);
                order.validate(msg.sender);
                transferInputTokens(order, msg.sender);
            }
        }
    }
```
It can be clearly seen that fees is injected on order instead of orders[i] and as order is copy of orders[i] it doesn't modifies orders[i]

## Impact
It causes fees to not be paid as the fees is not injected properly,so when _fill(resolvedOrders) is called it would only lead to fulfillment of the output tokens and not the fees.
## Code Snippet
https://github.com/sherlock-audit/2024-02-rubicon-finance/blob/11cac67919e8a1303b3a3177291b88c0c70bf03b/gladius-contracts-internal/src/reactors/BaseGladiusReactor.sol#L213
## Tool used

Manual Review

## Recommendation
make the following change
```solidity
function _prepare(ResolvedOrder[] memory orders) internal {
        uint256 ordersLength = orders.length;
        unchecked {
            for (uint256 i = 0; i < ordersLength; i++) {
               ResolvedOrder memory order = orders[i];
    @==>         _injectFees(orders[i]);
                order.validate(msg.sender);
                transferInputTokens(order, msg.sender);
            }
        }
    }
```
