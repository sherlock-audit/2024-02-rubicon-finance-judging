Boxy Boysenberry Vulture

medium

# Apply fee when false still adds a base fee

## Summary

When a token pair's fee applyFee is set to false, then the RubiconFeeController will still apply a base fee to the feeAmount. This is in direct contradiction to the code comments which state:

```solidity
/// @dev Fee controller, that's intended to be called by reactors.
///      * By default applies constant 'BASE_FEE' on output token.
///      * Dynamic pair-based fee can be enabled by calling 'setPairBasedFee'.
///      * Both dynamic and base fee can be disabled by setting 'applyFee' to false.
```

Take note of the last line, which states that the dynamic and base fee can be disabled by setting applyFee to false. 

## Vulnerability Detail

The vulnerability has to do with a if statement called in getFeeOutputs(). This if statement applies a base fee when the applyFee is set to false:

```solidity
uint256 feeAmount = fee.applyFee
    ? order.outputs[i].amount.mulDivUp(fee.fee, DENOM)
    : order.outputs[i].amount.mulDivUp(baseFee, DENOM);
```

The following spec test shows that a fee is applied when applyFee is set to false:

```solidity
// SPDX-License-Identifier: GPL-2.0-or-later
pragma solidity ^0.8.0;

import {Owned} from "solmate/src/auth/Owned.sol";
import {Test} from "forge-std/Test.sol";
import {InputToken, OutputToken, OrderInfo, ResolvedOrder, SignedOrder} from "../../src/base/ReactorStructs.sol";
import {NATIVE} from "../../src/lib/CurrencyLibrary.sol";
import {ProtocolFees} from "../../src/base/ProtocolFees.sol";
import {GladiusReactor} from "../../src/reactors/GladiusReactor.sol";
import {ResolvedOrderLib} from "../../src/lib/ResolvedOrderLib.sol";
import {MockERC20} from "../util/mock/MockERC20.sol";
import {OrderInfoBuilder} from "../util/OrderInfoBuilder.sol";
import {OutputsBuilder} from "../util/OutputsBuilder.sol";
import {MockProtocolFees} from "../util/mock/MockProtocolFees.sol";
import {PermitSignature} from "../util/PermitSignature.sol";
import {DeployPermit2} from "../util/DeployPermit2.sol";
import {IPermit2} from "permit2/src/interfaces/IPermit2.sol";
import {MockFillContract} from "../util/mock/MockFillContract.sol";
import {RubiconFeeController} from "../../src/fee-controllers/RubiconFeeController.sol";
import {ExclusiveDutchOrderReactor, ExclusiveDutchOrder, DutchInput, DutchOutput} from "../../src/reactors/ExclusiveDutchOrderReactor.sol";

contract RubiconFeeControllerTest is Test {
    using OrderInfoBuilder for OrderInfo;
    using ResolvedOrderLib for OrderInfo;

    event ProtocolFeeControllerSet(
        address oldFeeController,
        address newFeeController
    );

    address constant PROTOCOL_FEE_OWNER = address(11);
    address constant RECIPIENT = address(12);
    address constant SWAPPER = address(13);

    MockERC20 tokenIn;
    MockERC20 tokenOut;
    MockERC20 tokenOut2;
    MockProtocolFees fees;
    RubiconFeeController feeController;
    GladiusReactor reactor;

    function setUp() public {
        fees = new MockProtocolFees(PROTOCOL_FEE_OWNER);
        tokenIn = new MockERC20("Input", "IN", 18);
        tokenOut = new MockERC20("Output", "OUT", 18);
        tokenOut2 = new MockERC20("Output2", "OUT", 18);

        feeController = new RubiconFeeController();
        feeController.initialize(PROTOCOL_FEE_OWNER, RECIPIENT);

        reactor = new GladiusReactor();

        vm.startPrank(PROTOCOL_FEE_OWNER);
        feeController.setGladiusReactor(payable(address(reactor)));
        fees.setProtocolFeeController(address(feeController));
        vm.stopPrank();
    }

    function test_PairBasedFeeBug() public {
        uint feeBps = 10;

        ResolvedOrder memory order = createOrder(1 ether);
        /// @dev Apply pair-based fee.
        vm.prank(PROTOCOL_FEE_OWNER);

        feeController.setPairBasedFee(
            address(tokenIn),
            address(tokenOut),
            feeBps,
            false // AUDIT: applyFee set to false
        );

        OutputToken[] memory result = feeController.getFeeOutputs(order);

        // AUDIT: fee amount will be greater than zero. 
        require(result[0].amount > 0);
    }
 
    function createOrder(
        uint256 amount
    ) private view returns (ResolvedOrder memory) {
        OutputToken[] memory outputs = new OutputToken[](1);
        address outputToken = address(tokenOut);
        outputs[0] = OutputToken(outputToken, amount, SWAPPER);
        return
            ResolvedOrder({
                info: OrderInfoBuilder.init(address(0)),
                input: InputToken(tokenIn, 1 ether, 1 ether),
                outputs: outputs,
                sig: hex"00",
                hash: bytes32(0)
            });
    }
}
```

## Impact

A base fee will be applied to a trade even if the applyFee is set to false. This can lead to users having to pay for a fee when applyFee is set to false.


## Code Snippet

https://github.com/sherlock-audit/2024-02-rubicon-finance/blob/main/gladius-contracts-internal/src/fee-controllers/RubiconFeeController.sol?plain=1#L12-L15

https://github.com/sherlock-audit/2024-02-rubicon-finance/blob/main/gladius-contracts-internal/src/fee-controllers/RubiconFeeController.sol?plain=1#L81-L83

## Tool used

Manual Review

## Recommendation

When fee.applyFee is false, return order.outputs[i].amount.

```solidity
uint256 feeAmount = fee.applyFee
    ? order.outputs[i].amount.mulDivUp(fee.fee, DENOM)
    : order.outputs[i].amount;
```