Brave Teal Starling

high

# handleOverride is not done correctly on the resolvedOrder

## Summary
If the conditions of the overriding the exclusive filler is meet then handleOverride is executed on the resolved order which increases the output amounts by exclusivity override bps but currently it is not applied the order instead it is applied on the copy of the resolved order because of memory variable.

## Vulnerability Detail
Following is resolve function 
```solidity
function resolve(
        SignedOrder calldata signedOrder,
        uint256 quantity
    ) internal view override returns (ResolvedOrder memory resolvedOrder) {
        GladiusOrder memory order = abi.decode(
            signedOrder.order,
            (GladiusOrder)
        );

        _validateOrder(order);

        /// @dev Apply decay function.
        InputToken memory input = order.input.decay(
            order.decayStartTime,
            order.decayEndTime
        );
        OutputToken[] memory outputs = order.outputs.decay(
            order.decayStartTime,
            order.decayEndTime
        );

        /// @dev Apply partition function.
        (input, outputs) = quantity.partition(
            input,
            outputs,
            order.fillThreshold
        );

        resolvedOrder = ResolvedOrder({
            info: order.info,
            input: input,
            outputs: outputs,
            sig: signedOrder.sig,
            hash: order.hash()
        });
        resolvedOrder.handleOverride(
            order.exclusiveFiller,
            order.decayStartTime,
            order.exclusivityOverrideBps
        );
    }
```
now we will see the handleOverride function 
```solidity
/// @notice Applies exclusivity override to the resolved order if necessary
    /// @param order The order to apply exclusivity override to
    /// @param exclusive The exclusive address
    /// @param exclusivityEndTime The exclusivity end time
    /// @param exclusivityOverrideBps The exclusivity override BPS
    function handleOverride(
        ResolvedOrder memory order,
        address exclusive,
        uint256 exclusivityEndTime,
        uint256 exclusivityOverrideBps
    ) internal view {
        // if the filler has fill right, we proceed with the order as-is
        if (hasFillingRights(exclusive, exclusivityEndTime)) {
            return;
        }

        // if override is 0, then assume strict exclusivity so the order cannot be filled
        if (exclusivityOverrideBps == STRICT_EXCLUSIVITY) {
            revert NoExclusiveOverride();
        }

        // scale outputs by override amount
        OutputToken[] memory outputs = order.outputs;
        for (uint256 i = 0; i < outputs.length; ) {
            OutputToken memory output = outputs[i];
            output.amount = output.amount.mulDivDown(
                BPS + exclusivityOverrideBps,
                BPS
            );

            unchecked {
                i++;
            }
        }
    }
```
As can be seen from the comments it is used to apply override to the order passed to the function 
from the following lines it is clearly seen that output amount is increased by exclusivityOverrideBps but instead of increasing the order.outputs[0].amount it is increasing the output amount of its copy so effectively it is not increasing the amount of original resolved order output amount.
```solidity
 OutputToken[] memory outputs = order.outputs;
        for (uint256 i = 0; i < outputs.length; ) {
            OutputToken memory output = outputs[i];
            output.amount = output.amount.mulDivDown(
                BPS + exclusivityOverrideBps,
                BPS
            );

            unchecked {
                i++;
            }
        }
```

## Impact
Exclusivity Override bps is not applied correctly to the resolved order thus the filler is exempted from paying the extra amount.
## Code Snippet
https://github.com/sherlock-audit/2024-02-rubicon-finance/blob/11cac67919e8a1303b3a3177291b88c0c70bf03b/gladius-contracts-internal/src/lib/ExclusivityOverrideLib.sol#L45
## Tool used

Manual Review

## Recommendation
make the following change in the for loop
```solidity
 OutputToken[] memory outputs = order.outputs;
        for (uint256 i = 0; i < outputs.length; ) {
         
   @==>     order.outputs[i].amount = order.outputs[i].amount.mulDivDown(
                BPS + exclusivityOverrideBps,
                BPS
            );

            unchecked {
                i++;
            }
        }
```