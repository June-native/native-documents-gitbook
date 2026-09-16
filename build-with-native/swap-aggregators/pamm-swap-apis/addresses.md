# Address References

pAMM uses Native Router **V6** and Native RFQ Pool **V6**. See [addresses.md](../../../resources/addresses/README.md "mention") for the full V6 layout on every chain.

{% hint style="warning" %}
Always read `directSwapEnabled` onchain before sending a swap. The pair table below is a snapshot from 16 September 2026 and can change without a docs update.
{% endhint %}

## BNB Chain (chainId 56)

<table><thead><tr><th width="257">Contract</th><th>Address</th></tr></thead><tbody><tr><td>NativeRouter</td><td>0x1fDED89D98CBeADd96a109D28689c2638025dad3</td></tr><tr><td>NativeRFQPool</td><td>0x8Ac1009D5f13660CE7089901E51459142ad50793</td></tr><tr><td>WBNB</td><td>0xbb4CdB9CBd36B01bD1cBaEBF2De08d9173bc095c</td></tr></tbody></table>

<table><thead><tr><th>Pair</th><th>Engine</th><th>Direct swap</th><th>Notes</th></tr></thead><tbody><tr><td>QQQB – USDT</td><td>0xa02223bBc5F60D8FD0F42E2b5ab1FC3C37be7652</td><td>Enabled</td><td>Live for test.</td></tr><tr><td>WBNB – USDT</td><td>0x62c3E84789ef7aB1CB4b27eAb45649441880FCB6</td><td>Disabled</td><td></td></tr><tr><td>USDC – USDT</td><td>0x06a9F8e8967076BbcEF5facc25FA3767Cab7bebf</td><td>Disabled</td><td></td></tr></tbody></table>

## Other V6 chains

Arbitrum, Base, Monad, X Layer, Morph, and Robinhood Chain have the same V6 router/pool layout (see the [address page](../../../resources/addresses/README.md "mention")) but **no pAMM engines assigned** at this snapshot.

For Ethereum, Native is working with Titan and other builders to integrate pAMM.

## How to confirm a pair yourself

The pool cannot enumerate pairs. Seed token pairs, then keep those with `getEngine(a, b) != address(0)` and `engine.rfqPool() == pool`.

```typescript
const engine = await pool.getEngine(tokenA, tokenB); // zero = not listed
const pairKey = await pool.getPairKey(tokenA, tokenB);
const enabled = await pool.directSwapEnabled(pairKey);
const boundPool = await engine.rfqPool(); // must equal the NativeRFQPool you use
```

* `tokenA` / `tokenB` order does not matter. `address(0)` is normalized to WETH/WBNB on the pool.
* `pairKey` is `keccak256(abi.encode(token0, token1))` after WETH-normalize, with `token0 < token1`.
* Also require `router.isNativePools(pool) == true`.
