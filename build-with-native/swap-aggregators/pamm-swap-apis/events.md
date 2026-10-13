# Events

The **pool** emits fill events. The engine does **not** emit a fill event.

## `PAMMTrade` — listen to this for direct swaps

Emitted by `NativeRFQPool`.

```solidity
event PAMMTrade(
    address recipient,
    address sellerToken,
    address buyerToken,
    uint256 sellerTokenAmount,
    uint256 buyerTokenAmount,
    bytes16 quoteId,
    address signer
);
```

<table><thead><tr><th>Field</th><th>Direct <code>tradePAMM</code></th><th>RFQ path that used pAMM</th></tr></thead><tbody><tr><td><code>quoteId</code></td><td><code>bytes16(0)</code></td><td>RFQ quote id</td></tr><tr><td><code>signer</code></td><td>Pair <code>pammTrader</code></td><td>Pair <code>pammTrader</code></td></tr><tr><td><code>buyerTokenAmount</code></td><td>Actual amount out</td><td>Actual amount out</td></tr></tbody></table>

Topic0:

`PAMMTrade(address,address,address,uint256,uint256,bytes16,address)`

`0xd9a78419f4f94739684d4bed95a1eb594d0b5f14272ef96a546fff2cc7d0b65d`

None of the fields are indexed — filter by pool address, then decode.

For native legs, `sellerToken` / `buyerToken` hold the wrapped token (WETH/WBNB), never `address(0)`.

Direct pAMM **does not** emit `RFQTrade`. For a given trade, a pAMM fill and an RFQ fill are mutually exclusive.

## `PAMMFallback` — RFQ path only

```solidity
event PAMMFallback(bytes16 indexed quoteId, address indexed engine, bytes4 reason);
```

Emitted on the RFQ path when the pAMM engine attempt reverts. The trade is then settled as an `RFQTrade`. Direct `tradePAMM` swaps never emit it. Compute topic0 as the keccak256 of `PAMMFallback(bytes16,address,bytes4)`.

## Config events (ops / indexing)

```solidity
event EngineUpdated(bytes32 indexed pairKey, address indexed engine, address indexed pammTrader);
event DirectSwapEnabledUpdated(bytes32 indexed pairKey, bool enabled);
```

`DirectSwapEnabledUpdated` is emitted only when the flag value changes, while `EngineUpdated` is always emitted. Cache the flag from `DirectSwapEnabledUpdated`, and re-read `directSwapEnabled` onchain whenever `EngineUpdated` fires.
