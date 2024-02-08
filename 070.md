Refined Green Coyote

high

# When an order is exactly matched, a buyer can end up paying more than his max amount due to execlusivityOverrideBps

## Summary
A buyer can end up paying more than his max amount due to `execlusivityOverrideBps` being applied after validating against the invariant `output.amount` set by the buyer.

## Vulnerability Detail
Let's consider the following scenario(graph taken from `test/reactors/GladiusReactor.t.sol`):
```javascript
    ///       ----------------------------
    ///      |         X/Y pair           |
    ///       ----------------------------
    ///Seller| (X/Y (sell 100) (buy 200)) | <-----
    ///Buyer |                            |       | match
    ///      | (Y/X (sell 200) (buy 100)) | <-----
    ///       ----------------------------
```
- Seller sells 100 USDC, seller gets 200 DAI.
- Buyer sells 200 DAI, buyer gets 100 USDC.

The problem here is, which is unaccounted for in the test suite due to modified mocks being used instead the real contract, is that the buyer will end up spending more than `200 DAI` due to `handleOverride` being applied after the validation that happens in `PartialFillLib.partition`:
```javascript
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
->        OutputToken[] memory outputs = order.outputs;
->        for (uint256 i = 0; i < outputs.length; ) {
->            OutputToken memory output = outputs[i];
->            output.amount = output.amount.mulDivDown(
->                BPS + exclusivityOverrideBps,
->                BPS
->            );
->            unchecked {
->                i++;
            }
        }
    }

```

In `handleOverride`, the `output.amount` gets increased with the `exclusivityOverrideBps` if the filler has no fill right. This would normally be no problem if it got validated, however, this increase of `exclusivityOverrideBps` happens **after** the validation of the partition:
```javascript
    function partition(
        uint256 quantity,
        InputToken memory input,
        OutputToken[] memory output,
        uint256 fillThreshold
    ) internal pure returns (InputToken memory, OutputToken[] memory) {
        _validateThreshold(fillThreshold, input.amount);

        uint256 outPart = quantity.mulDivUp(output[0].amount, input.amount);

        _validatePartition(
            quantity,
            outPart,
            input.amount,
            output[0].amount,
            fillThreshold
        );

        // Mutate amounts in structs.
        input.amount = quantity;
        output[0].amount = outPart;

        return (input, output);
    }
        function _validatePartition(
        uint256 _quantity,
        uint256 _outPart,
        uint256 _initIn,
        uint256 _initOut,
        uint256 _fillThreshold
    ) internal pure {
->      if (_quantity > _initIn || _outPart > _initOut)
            revert PartialFillOverflow();
        if (_quantity == 0 || _outPart == 0) revert PartialFillUnderflow();
        if (_quantity < _fillThreshold) revert QuantityLtThreshold();
    }
```
This means that the `output.amount + overrideBps applied`,  nullifies this invariant check that happens above:
- `if (_quantity > _initIn || _outPart > _initOut) revert PartialFillOverflow();`

`outPart`, which gets checked against `_initOut`(output.amount), is being assured that its not more than the output.amount.
After this check, `output.amount` gets set to `outPart`.

Unfortunately this validation happens before applying the `excessOverrideBps`, which means that this check gets broken and the user ends up paying more than `output.amount`.
## Impact
To continue with our example above, lets say our seller is a malicious user who has put the `execlusivityOverrideBps` at high amount.
The buyer would in this case expect his invariant of `output.amount` to not be exceeded. 
However, if there is `exclusivityOverrideBps` set, the buyer will pay more than `output.amount`, breaking this critical invariant that should never be broken. If a user puts a limit order to buy 100 USDC using 200 DAI, it should at max take 200 DAI, not 200 DAI + overrideBps.

One might say a person will need to approve `permit2` the exact amount before executing each trade, however, in reality, people just approve an `x amount` of tokens  to `permit2` so they can efficiently trade. 

Furthermore, one might say, the buyer can see the `exclusivityOverrideBps`, which is true, however, the `exclusivityOverrideBps` added should never exceed `amount.output`, which is the case since the project checks for this invariant to hold inside `_validatePartition`.
## Code Snippet
https://github.com/sherlock-audit/2024-02-rubicon-finance/blob/main/gladius-contracts-internal/src/lib/PartialFillLib.sol#L119-L130
## Tool used
Manual Review
## Recommendation
Apply the `execlusivityOverrideBps` before the `PartialFillLib.partition` function.