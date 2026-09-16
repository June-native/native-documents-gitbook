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

Direct pAMM **does not** emit `RFQTrade`. A successful pAMM fill and an RFQ fill are mutually exclusive.

## Events you can ignore on the direct path

<table><thead><tr><th>Event</th><th>When it fires</th></tr></thead><tbody><tr><td><code>PAMMFallback(bytes16 quoteId, address engine, bytes4 reason)</code></td><td>RFQ tried pAMM and the engine reverted, then RFQ continued. <strong>Not</strong> emitted by <code>tradePAMM</code>.</td></tr><tr><td><code>RFQTrade</code></td><td>RFQ fill (<code>tradeRFQT</code>). Direct swaps never emit this.</td></tr></tbody></table>

`PAMMFallback` topic0: `0x2a3c7ab9d4eaf2c960f0510c29e51e0455964d9e6a549ac28cee63c47f970146` (`quoteId` and `engine` are indexed).

## Config events (ops / indexing)

```solidity
event EngineUpdated(bytes32 indexed pairKey, address indexed engine, address indexed pammTrader);
event DirectSwapEnabledUpdated(bytes32 indexed pairKey, bool enabled);
```

Use `DirectSwapEnabledUpdated` to cache the live flag instead of polling every block.
