Clever Rusty Starfish

medium

# Users can grief fillers by set malicious `ValidationContract`.

## Summary

When a user creates a new order, he can pass in an `additionalValidationContract`  depending on his preference. When a filler executes the order, it calls the `validate` function in that contract to do some check. However, the problem is that this `ValidationContract` can be specified by the creator of the order, making it possible for gas-griefing attack.

## Vulnerability Detail

When fillers try to [`execute`](https://github.com/sherlock-audit/2024-02-rubicon-finance/blob/main/gladius-contracts-internal/src/reactors/BaseGladiusReactor.sol#L53-L63) a order, `order.validate(msg.sender)` is called:

    function execute(
        SignedOrder calldata order
    ) external payable override nonReentrant {
        ResolvedOrder[] memory resolvedOrders = new ResolvedOrder[](1);
        resolvedOrders[0] = resolve(order);

        _prepare(resolvedOrders);
        _fill(resolvedOrders);
    }

    function _prepare(ResolvedOrder[] memory orders) internal {
        uint256 ordersLength = orders.length;
        unchecked {
            for (uint256 i = 0; i < ordersLength; i++) {
                ResolvedOrder memory order = orders[i];
                _injectFees(order);
                order.validate(msg.sender);
                transferInputTokens(order, msg.sender);
            }
        }
    }

    //@Audit /lib/ResolvedOrderLib.sol
    function validate(
        ResolvedOrder memory resolvedOrder,
        address filler
    ) internal view {
        if (address(this) != address(resolvedOrder.info.reactor)) {
            revert InvalidReactor();
        }

        if (block.timestamp > resolvedOrder.info.deadline) {
            revert DeadlinePassed();
        }

        if (
            address(resolvedOrder.info.additionalValidationContract) !=
            address(0)
        ) {
            resolvedOrder.info.additionalValidationContract.validate(
                filler,
                resolvedOrder
            );
        }
    }

Attack path:

1. Attacker makes a tiny swap order, waiting for fillers to take it.
2. Attacker notices the transaction of filler in the mempool.
3. Attacker frongrun the tx, triggers self-destruct and redeploy malicious bytecode on the same address.
4. Filler`s tx gets executed, resulting in a significant loss for the filler.

(Note: such attack doesn't fully rely on frontrun. An attacker can also "guess" the block where the transaction would be executed by offering a good price and make the logic of `validate` change at that block)

## Impact

Fillers may suffer great loss in ETH.

## Code Snippet

https://github.com/sherlock-audit/2024-02-rubicon-finance/blob/main/gladius-contracts-internal/src/reactors/BaseGladiusReactor.sol#L53-L63

## Tool used

Manual Review

## Recommendation

Verify if `FillerValidation` is a trusted address which is controlled by protocol team.
