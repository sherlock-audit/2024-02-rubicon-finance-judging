Tricky Goldenrod Moose

high

# The `owner` address has never been set wich will cause the `auth` modifier to revert

## Summary
The `owner` address has never been set which will cause functions to break.

## Vulnerability Detail
Contract DSAuth has an address `owner` this should be used to check whether the `msg.sender == owner`using the function `isAuthorized`. The problem lies in owner never being set which causes `src == owner` to never be true. The owner can also not be changed because it has the modifier `auth` attached to it.
## Impact
Since `owner` will always equal to `0x0` anybody who calls a function that has the `auth` modifier will always revert since `msg.sender != 0x0`.
## Code Snippet
```solidity
contract DSAuth is DSAuthEvents {
 @> address public owner;

    error Unauthorized();

    function setOwner(address owner_) external auth {
        owner = owner_;
        emit LogSetOwner(owner);
    }

    modifier auth() {
@>    if (!isAuthorized(msg.sender)) revert Unauthorized();
        _;
    }

    function isAuthorized(address src) internal view returns (bool) {
@>    if (src == owner) {
            return true;
        } else {
            return false;
        }
    }
}
```
https://github.com/sherlock-audit/2024-02-rubicon-finance/blob/main/gladius-contracts-internal/src/lib/DSAuth.sol#L12-L32
## Tool used

Manual Review, foundry

## Recommendation
Declare the owner at the start.
