Elegant Tan Wallaby

high

# `execute` transactions can be reverted by a malicious user.

## Summary
`execute`, `executeWithCallback`, `executeBatch`, and `executeBatchWithCallback` transactions can be made to revert by an attacker.

## Vulnerability Detail
Please read this blog first by Trust Security - https://www.trust-security.xyz/post/permission-denied

The following is a little similar to the issue above.

When a filler calls the `execute` transaction, its parameter `order` can be seen by anyone in the mempool. 

```solidity
    function execute(
        SignedOrder calldata order
    ) external payable override nonReentrant {
```

This parameter is used to call the function `permit2.permitWitnessTransferFrom` in the function `transferInputTokens` when the execution in the function `execute` enters the `_prepare` code block. 

```solidity
    function transferInputTokens(
        ResolvedOrder memory order,
        address to
    ) internal override {
        permit2.permitWitnessTransferFrom(
            order.toPermit(),
            order.transferDetails(to),
            order.info.swapper,
            order.hash,
            PartialFillLib.PERMIT2_ORDER_TYPE,
            order.sig
        );
    }
```

A malicious user upon seeing the `execute` transaction, knows the values of the `order` parameter. They can use it to call the `permit2.permitWitnessTransferFrom` themselves. They would frontrun the `execute` function. Once the attacker does it the nonce associated with the transaction cannot be used again.  If the nonce is used again in another transaction, that transaction will revert. Hence, the original `execute` transaction by the filler would revert. The `execute` function would not be able to call its sub-functions like the `_fill` function which is responsible for output token transfers.

The malicious user can keep doing this as long as they want, causing a Denial of Service attack.

Also, the Notion document says the [following](https://rubicondefi.notion.site/Rubicon-Gladius-Integration-Guide-ee55cbe3ccdf4b88a6780d8ef090599f#:~:text=Offers%20are%20submitted,and%20gas%20fees.):

> Offers are submitted as off-chain intents. What this means in practice, is that the signer of the order says “I would like to execute this exact trade, valid for this amount of time”. An onchain action can only occur if an activist in the system wants to fill their trade exactly while paying any applicable protocol fees and gas fees.

This means that the `order` parameter can be seen much more before the `execute` transaction is called by the filler, making this attack more likely to happen.

## Impact
A denial of service attack due to which users are never able to get their intended output tokens back.

## Code Snippet
https://github.com/sherlock-audit/2024-02-rubicon-finance/blob/11cac67919e8a1303b3a3177291b88c0c70bf03b/gladius-contracts-internal/src/reactors/BaseGladiusReactor.sol#L53

https://github.com/sherlock-audit/2024-02-rubicon-finance/blob/11cac67919e8a1303b3a3177291b88c0c70bf03b/gladius-contracts-internal/src/reactors/GladiusReactor.sol#L115

## Tool used

Manual Review

## Recommendation 
