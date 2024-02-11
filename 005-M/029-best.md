Attractive Iron Bear

medium

# `PartialFillLib::partition()` unexpectedly reverts with `PartialFillOverflow` error, due to rounding up the output tokens.

## Summary
GladiusReactor.sol#resolve( order , quantity ) is using `partition()` on input/output tokens, any ask order can be partially filled with input tokens of amount `quantity` such that: `fillThreshold <= quantity <= order.input.amount`.

In `partition()`, even though the filled `quantity` is same as `order.input.amount`, it is expected the output token amount(which is outpart) to be same as `output[0].amount`. The issue its not, because `mulDivUp` rounds up by adding 1 to it, which means `outPart` always > `output[0].amount` for such case. 

Check out the comments below; 
https://github.com/transmissions11/solmate/blob/c892309933b25c03d32b1b0d674df7ae292ba925/src/utils/FixedPointMathLib.sol#L66


```solidity
    function _validatePartition(
        uint256 _quantity,
        uint256 _outPart,
        uint256 _initIn,
        uint256 _initOut,
        uint256 _fillThreshold
    ) internal pure {
        if (_quantity > _initIn || _outPart > _initOut) // <= @audit if quantity==inputAmount, _outPart always > _initOut, hence revert
            revert PartialFillOverflow();
        ...SNIP...
    }
```

## Vulnerability Detail
See summary. 
## Impact
Unable to fill the order via above even it expected to be filled. Note that order can also be filled via `resolve(order)` but until the filler switch to that, someone may end up executing the order already via `resolve(order)`. Fillers may end up losing their winning arbitrage. 

## Code Snippet
https://github.com/sherlock-audit/2024-02-rubicon-finance/blob/11cac67919e8a1303b3a3177291b88c0c70bf03b/gladius-contracts-internal/src/lib/PartialFillLib.sol#L93-L101
## Tool used

Manual Review

## Recommendation
```diff
    function _validatePartition(
        uint256 _quantity,
        uint256 _outPart,
        uint256 _initIn,
        uint256 _initOut,
        uint256 _fillThreshold
    ) internal pure {
-       if (_quantity > _initIn || _outPart > _initOut)
+       if (_quantity > _initIn)
            revert PartialFillOverflow();
        if (_quantity == 0 || _outPart == 0) revert PartialFillUnderflow();
        if (_quantity < _fillThreshold) revert QuantityLtThreshold();
    }
```