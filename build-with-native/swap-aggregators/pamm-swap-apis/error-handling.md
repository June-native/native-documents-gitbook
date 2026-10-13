# Error Handling

`tradePAMM` and `getQuote` revert on failure. There is no HTTP error envelope on this path.

## Swap and quote reverts

<table><thead><tr><th width="292">Error</th><th>Meaning</th></tr></thead><tbody><tr><td><code>DirectSwapDisabled</code></td><td><code>directSwapEnabled</code> is off for the pair, or the pair has no engine assigned. A quote may still succeed if an engine exists.</td></tr><tr><td><code>InvalidNativePool</code></td><td>Pool not whitelisted on this router.</td></tr><tr><td><code>QuoteExpired</code></td><td><code>deadlineTimestamp</code> passed.</td></tr><tr><td><code>PriceExpired</code></td><td>The engine price is no longer valid. V2 engines expire by block number (<code>validUntil</code>); V1 engines expire by timestamp. Re-quote and retry.</td></tr><tr><td><code>PriceOutsideAllowedRange</code></td><td>Fill is below <code>amountOutMinimum</code> (the engine's slippage check). Widen slippage, reduce size, or re-quote.</td></tr><tr><td><code>InsufficientDepth</code></td><td>Size larger than remaining ladder/curve.</td></tr><tr><td><code>BelowMinimumTradeSize</code></td><td>Under <code>minBaseInputRaw</code> / <code>minQuoteInputRaw</code>, or the net amount rounds to zero.</td></tr><tr><td><code>UnsupportedPair</code></td><td>The sell/buy tokens are not this engine's <code>baseAsset()</code> / <code>quoteAsset()</code>.</td></tr><tr><td><code>InvalidPrice</code> / <code>ZeroAmountOut</code></td><td>Zero input amount, or the quote resolves to zero output.</td></tr><tr><td><code>InsufficientVaultCash</code></td><td>Quote-time only. The treasury does not hold enough of the buy token to pay the output.</td></tr><tr><td><code>PositionLimitExceeded</code></td><td>The pair's inventory cap would be exceeded. Try a smaller size or the opposite direction.</td></tr><tr><td><code>ZeroInput</code></td><td>Router rejected the call: zero recipient, <code>sellerToken == buyerToken</code>, or zero amount.</td></tr><tr><td><code>UnexpectedMsgValue</code></td><td><code>msg.value &gt; 0</code> while selling an ERC-20. Send <code>msg.value = 0</code>.</td></tr><tr><td><code>InvalidAmount</code></td><td>Selling native and <code>msg.value != sellerTokenAmount</code>, or the received amount differs from the requested amount (for example a fee-on-transfer token).</td></tr><tr><td><code>EnforcedPause</code></td><td>The router is paused.</td></tr></tbody></table>

{% hint style="info" %}
Selling native with `sellerToken = address(0)` but `msg.value = 0` is treated as an ERC-20 sell and fails with an unnamed token-transfer revert. Always send `msg.value = sellerTokenAmount` when selling native.
{% endhint %}

{% hint style="info" %}
`NotEnoughAmountOut` is raised on the RFQ path (`tradeRFQT`), not on direct pAMM swaps. `InvalidEngine` is raised only by admin calls that configure an engine.
{% endhint %}
