# Issue M-1: Pairs with "MAX_FEE" can revert due to rounding inconsistencies 

Source: https://github.com/sherlock-audit/2024-02-rubicon-finance-judging/issues/51 

## Found by 
KingNFT, Kow, cawfree, mstpr-brainbot
## Summary
If the pair has set the max fee by the fee controller admin which is "1_000" then depending on the amount to be swapped, the tx can revert due to rounding error. 
## Vulnerability Detail
When the fee amount is calculated inside the RubiconFeeController, the fee amount is rounded down. 
```solidity
uint256 feeAmount = fee.applyFee
                ? order.outputs[i].amount.mulDivUp(fee.fee, DENOM)
                : order.outputs[i].amount.mulDivUp(baseFee, DENOM);
```

then, ProtocolFees abstract contract will do a double check on the fee taken as follows:

```solidity
if (feeOutput.amount > tokenValue.mulDivDown(MAX_FEE, DENOM)) {
                revert FeeTooLarge(
                    feeOutput.token,
                    feeOutput.amount,
                    feeOutput.recipient
                );
            }
```
As we can see, it uses mulDivDown, so if the calculation in the FeeController rounds up, the transaction will revert.

**Textual PoC:**
Suppose the fee pair is set to "1_000" for tokens A and B.
Alice sends an order to sell "111111111111111111111" (111.11 in 18 decimals) token A for token B.

Within the fee controller, the fee amount will be calculated as:
111111111111111111111 * 1000 / 100_000 (roundUp) = 1111111111111111112

Subsequently, during execution, within the ProtocolFees contract, the maximum fee amount will be computed as:
111111111111111111111 * 1000 / 100_000 (roundDown) = 1111111111111111111

Consequently, the transaction will revert because 1111111111111111111 > 1111111111111111112.

**Coded PoC:**
```solidity
// forge test --match-contract GladiusReactorTest --match-test test_FeesRounding -vv
    function test_FeesRounding(uint amount) external {
        // @dev there will be plenty of values reverting this test. 
        vm.assume(amount <= type(uint128).max);
        vm.assume(amount >= 1e6);

        uint DENOM = 100_000;
        uint FEE = 1_000;

        uint resultDown = FixedPointMathLib.mulDivDown(amount, FEE, DENOM);
        uint resultUp = FixedPointMathLib.mulDivUp(amount, FEE, DENOM);

        assertEq(resultDown, resultUp);
    }
```
## Impact
As stated in README:
**Fee Controller Can DOS Trading Activity. Note, that, as said above, the resulting output shouldn't overflow MAX_FEE, but other possibilities of reverts are known/acceptable.** Any fee setting in range 0<MAX_FEE should not revert and if it reverts then its acceptable. Hence, I'll label this as medium.
## Code Snippet
https://github.com/sherlock-audit/2024-02-rubicon-finance/blob/11cac67919e8a1303b3a3177291b88c0c70bf03b/gladius-contracts-internal/src/fee-controllers/RubiconFeeController.sol#L63-L116

https://github.com/sherlock-audit/2024-02-rubicon-finance/blob/11cac67919e8a1303b3a3177291b88c0c70bf03b/gladius-contracts-internal/src/base/ProtocolFees.sol#L39-L105
## Tool used

Manual Review

## Recommendation



## Discussion

**sherlock-admin**

1 comment(s) were left on this issue during the judging contest.

**0xAadi** commented:
> 



# Issue M-2: When an order is exactly matched, a buyer can end up paying more than his max amount due to execlusivityOverrideBps 

Source: https://github.com/sherlock-audit/2024-02-rubicon-finance-judging/issues/70 

## Found by 
Fassi\_Security
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



## Discussion

**sherlock-admin**

1 comment(s) were left on this issue during the judging contest.

**0xAadi** commented:
> 



