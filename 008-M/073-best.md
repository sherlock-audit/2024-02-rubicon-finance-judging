Puny Aqua Ladybug

medium

# [H] `RubiconFeeController::getFeeOutputs` incorrectly creates feeOutput tokens, thus adding duplicates to the `order.outputs`

high

## Summary
The problem is that `order.outputs` and `feeOutputs` arrays contain the same tokens, and in the end we add `feeOutputs` to the `order.outputs` and pay out the same output twice.

## Vulnerability Detail
As we can see, there is a check for the tokens with a special fee [`if (feeAmount != 0)`](https://github.com/sherlock-audit/2024-02-rubicon-finance/blob/main/gladius-contracts-internal/src/fee-controllers/RubiconFeeController.sol#L86). 
Then it starts looping through the results arrays with the condition `j = 0; j < feeCount; j++)`. The first interaction will be skipped cause `feeCount` is zero. Therefore, `bool found` will stay false and enter `if (!found)` clause. And we add current `tokenOut` to the `result` array.
<details>
<summary>Code for the text above:</summary>

```solidity
            if (feeAmount != 0) {
                bool found;

                for (uint256 j = 0; j < feeCount; ++j) {
                    OutputToken memory feeOutput = result[j];

                    if (feeOutput.token == tokenOut) {
                        found = true;
                        feeOutput.amount += feeAmount;
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
```

</details>

Then we return `results` array to the `ProtocolFees::_injectFees` function. I know this contract is out of scope, but `BaseGladiusReactor` inherits from it and implements `_injectFees` function.
At some point at [L75](https://github.com/sherlock-audit/2024-02-rubicon-finance/blob/main/gladius-contracts-internal/src/base/ProtocolFees.sol#L75) there is a check that `feeOutput` token is in `outputs` array. But the function only increments token's value after it.
And in the end of the `_injectFees` function we add `feeOutput` token to the `newOutputs` array. `newOutputs` array is created at L50 and contains all the tokens from `order.outputs` (in this implementation only one token). Since `feeOutput` is the same token, now `newOutputs` contains the same token twice. 
<details>
<summary>Snippet from `_injectFees`</summary>

```solidity
      function _injectFees(ResolvedOrder memory order) internal view {
        if (address(feeController) == address(0)) {
            return;
        }

        OutputToken[] memory feeOutputs = feeController.getFeeOutputs(order);
        uint256 outputsLength = order.outputs.length;
        uint256 feeOutputsLength = feeOutputs.length;

        // apply fee outputs
        // fill new outputs with old outputs
        OutputToken[] memory newOutputs = new OutputToken[](
            outputsLength + feeOutputsLength
        );

        unchecked {
            for (uint256 i = 0; i < outputsLength; i++) {
                newOutputs[i] = order.outputs[i];
            }
        }

        for (uint256 i = 0; i < feeOutputsLength; ) {
            OutputToken memory feeOutput = feeOutputs[i];
            // assert no duplicates
            unchecked {
                for (uint256 j = 0; j < i; j++) {
                    if (feeOutput.token == feeOutputs[j].token) {
                        revert DuplicateFeeOutput(feeOutput.token);
                    }
                }
            }

            // assert not greater than MAX_FEE
            uint256 tokenValue;
            for (uint256 j = 0; j < outputsLength; ) {
                OutputToken memory output = order.outputs[j];
                if (output.token == feeOutput.token) {
                    tokenValue += output.amount;
                }
                unchecked {
                    j++;
                }
            }

            // allow fee on input token as well
            if (address(order.input.token) == feeOutput.token) {
                tokenValue += order.input.amount;
            }

            if (tokenValue == 0) revert InvalidFeeToken(feeOutput.token);
	    
            if (feeOutput.amount > tokenValue.mulDivDown(MAX_FEE, DENOM)) {
                revert FeeTooLarge(
                    feeOutput.token,
                    feeOutput.amount,
                    feeOutput.recipient
                );
            }
            newOutputs[outputsLength + i] = feeOutput;

            unchecked {
                i++;
            }
        }

        order.outputs = newOutputs;
    }
```

</details>

As we continue to go through the function till we reach the transfer of the output tokens inside `transferFill` function, there are no checks to if there are duplicates inside `order.outputs` and protocol ends up sending the same token amount twice.

## Impact
Contract loses money, cause it sends out output tokens twice for the swap.

## Code Snippet
[`RubiconFeeController::getFeeOutputs`](https://github.com/sherlock-audit/2024-02-rubicon-finance/blob/main/gladius-contracts-internal/src/fee-controllers/RubiconFeeController.sol#L63C5-L107C10)
[`ProtocolFees::_injectFees`](https://github.com/sherlock-audit/2024-02-rubicon-finance/blob/main/gladius-contracts-internal/src/base/ProtocolFees.sol#L39C5-L105C6)
[`BaseGladiusReactor::_fill`](https://github.com/sherlock-audit/2024-02-rubicon-finance/blob/main/gladius-contracts-internal/src/reactors/BaseGladiusReactor.sol#L222C1-L243C10)

## Tool used

Manual Review

## Recommendation
In any of the functions add a check that `feeOutput` token is inside the `outputs` array, and if yes, then change the amount for that particular token.
