Macho Maroon Parrot

medium

# Token with low decimals can have problems on fee and partial filling calculations

## Summary
Fee calculations and partial fill calculations can be off when working with tokens with low decimals like 0-1-2. 
## Vulnerability Detail
When calculating the fee to be deducted from an order, the following code is used:
```solidity
uint256 feeAmount = fee.applyFee
                ? order.outputs[i].amount.mulDivUp(fee.fee, DENOM)
                : order.outputs[i].amount.mulDivUp(baseFee, DENOM);
```
Let's assume the output token is EURS, which has 2 decimals, and the output amount is 10 * 1e2 EURS (10 EURS). The baseFee is set to "10" as default. The calculation for the feeAmount will be:
10 * 1e2 * 10 / 100_000 = 1, due to mulDivUp rounding the "0" to "1". Hence, the fee will be calculated as 1 EURS. However, the baseFee was 10, and the DENOM was 100_000, so the fee to be taken normally would be 0.1. But due to rounding in Solidity, the fee taken will be 1 EURS, which is 10x more.

The protocol team can set the fee tier to the maximum, such as 1000 (1%), to address the issue. However, this would mean that the protocol team would be taking a 1% fee just to mitigate the rounding error.

The same issue can also occur in the partial filling of a token like EURS. This is how the portioned output amount is calculated given a partial fill:
`uint256 outPart = quantity.mulDivUp(output[0].amount, input.amount);`

Assuming the quantity is 1e18, the total input amount is 1001 * 1e18, and the output amount is 100. The outPart will be calculated as:
1e18 * 100 / 1001 * 1e18 = 0, which rounds up to 1. This means if a filler fills this trade with this quantity, they will incur losses because outPart 1 is not the correct amount; it should have been 0.1. But due to Solidity rounding down, it is 10x higher. Fillers can choose not to fill this to avoid this loss, but that would mean that partial orders of such trades will not be possible.
## Impact
As stated in the README the code should work with any token with any decimals. There are such liquid tokens exists with low decimals such as; EURS, TEL, GLD and integrating those tokens with GladiusReactor can not work on some cases described in the details section. Considering that, I'll label this as medium.
## Code Snippet
https://github.com/sherlock-audit/2024-02-rubicon-finance/blob/11cac67919e8a1303b3a3177291b88c0c70bf03b/gladius-contracts-internal/src/fee-controllers/RubiconFeeController.sol#L63-L116

https://github.com/sherlock-audit/2024-02-rubicon-finance/blob/11cac67919e8a1303b3a3177291b88c0c70bf03b/gladius-contracts-internal/src/lib/PartialFillLib.sol#L85-L108
## Tool used

Manual Review

## Recommendation
