Dapper Iron Fox

medium

# Orders with equal `decayStartTime` and `decayEndTime` benefit the filler instead of swapper

## Summary

GladiusReactor uses a linear decay function to adjust the input and output amounts within the specified time range between `decayStartTime` and `decayEndTime`. An edge case arises when `decayStartTime` and `decayEndTime` are set to the same value, but `startAmount` and `endAmount` differ. In such a scenario, the trade's final amount defaults to `endAmount`, which benefits the filler over the swapper.

## Vulnerability Detail

As stated in the Summary.

This issue was actually fixed in the latest UniswapX in this PR https://github.com/Uniswap/UniswapX/pull/194, however it still exists in this codebase.

## Impact

Swappers might be unaware of this specific edge case, potentially leading them to unintentionally place orders that favor the filler instead of benefiting themselves.

## Code Snippet

- https://github.com/sherlock-audit/2024-02-rubicon-finance/blob/main/gladius-contracts-internal/src/lib/DutchDecayLib.sol#L32-L35
- https://github.com/Uniswap/UniswapX/pull/194

DutchDecayLib.sol
```solidity
    function decay(
        uint256 startAmount,
        uint256 endAmount,
        uint256 decayStartTime,
        uint256 decayEndTime
    ) internal view returns (uint256 decayedAmount) {
>       if (decayEndTime < decayStartTime) {
            revert EndTimeBeforeStartTime();
>       } else if (decayEndTime <= block.timestamp) {
            decayedAmount = endAmount;
        } else if (decayStartTime >= block.timestamp) {
            decayedAmount = startAmount;
        } else {
            ...

```

## Tool used

VSCode

## Recommendation

Disable dutch orders with zero duration to avoid ambiguity.

```diff
     ) internal view returns (uint256 decayedAmount) {
-        if (decayEndTime < decayStartTime) {
+        if (decayEndTime <= decayStartTime) {
             revert EndTimeBeforeStartTime();
         } else if (decayEndTime <= block.timestamp) {

```