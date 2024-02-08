Brave Teal Starling

medium

# Fee updated on the memory variable instead on the original result

## Summary
Fee is increased on the copy of the output token instead on the original one.
## Vulnerability Detail
In the following code instead of increasing the fee on the result array it is increased on its copy
```solidity
 function getFeeOutputs(
        ResolvedOrder memory order
    ) external view override returns (OutputToken[] memory result) {	
        /// @notice Right now the length is enforced by
        ///         'GladiusReactor' to be equal to 1.
        result = new OutputToken[](order.outputs.length);

        address tokenIn = address(order.input.token);
        uint256 feeCount;

        for (uint256 i = 0; i < order.outputs.length; ++i) {
            /// @dev Fee will be in the 'tokenOut' form.
            address tokenOut = order.outputs[i].token;

            PairBasedFee memory fee = fees[
                getPairHash(address(tokenIn), tokenOut)
            ];

            uint256 feeAmount = fee.applyFee
                ? order.outputs[i].amount.mulDivUp(fee.fee, DENOM)
                : order.outputs[i].amount.mulDivUp(baseFee, DENOM);

            /// @dev If fee is applied to pair.
            if (feeAmount != 0) {
                bool found;

                for (uint256 j = 0; j < feeCount; ++j) {
      @==>       OutputToken memory feeOutput = result[j];

                    if (feeOutput.token == tokenOut) {
                        found = true;
          @==>    feeOutput.amount += feeAmount;
                    }
                }

                if (!found) {
                    result[feeCount] = OutputToken({
                        token: tokenOut,
                        amount: feeAmount,
                        recipient: feeRecipient
                    });
                    feeCount++;
                }
            }
        }

        assembly {
            // update array size to the actual number of unique fee outputs pairs
            // since the array was initialized with an upper bound of the total number of outputs
            // note: this leaves a few unused memory slots, but free memory pointer
            // still points to the next fresh piece of memory
            mstore(result, feeCount)
        }
    }
```
## Impact
Fee is not correctly updated
## Code Snippet
https://github.com/sherlock-audit/2024-02-rubicon-finance/blob/11cac67919e8a1303b3a3177291b88c0c70bf03b/gladius-contracts-internal/src/fee-controllers/RubiconFeeController.sol#L94
## Tool used

Manual Review

## Recommendation
make the following change
```solidity
 if (feeAmount != 0) {
                bool found;

                for (uint256 j = 0; j < feeCount; ++j) {
           OutputToken memory feeOutput = result[j];

                    if (feeOutput.token == tokenOut) {
                        found = true;
          @==>    result[j].amount += feeAmount;
                    }
                }
```