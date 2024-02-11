Macho Maroon Parrot

medium

# Pairs with "MAX_FEE" can revert due to rounding inconsistencies

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
