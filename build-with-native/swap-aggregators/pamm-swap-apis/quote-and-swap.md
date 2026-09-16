# Quote and Swap

## Discover the engine

The pool cannot enumerate pairs. Seed token pairs, then keep those with `getEngine(a, b) != address(0)` and `engine.rfqPool() == pool`. See [addresses.md](addresses.md "mention") for deployed engines and the live `directSwapEnabled` check.

## Quote — `PropAMMEngine.getQuote`

```solidity
function getQuote(
    address sellerToken,
    address buyerToken,
    uint256 sellerTokenAmount
) external view returns (uint256 amountOut);
```

* Amounts are **raw token units** (respect decimals).
* Output is **after engine fees**. The call does not consume depth.
* For native gas token (ETH/BNB), pass the **wrapped** address to `getQuote`. Never pass `address(0)` on the engine.
* Reverts when price is expired, depth is insufficient, size is below `minBaseInputRaw` / `minQuoteInputRaw`, or the direction is not this engine's base/quote pair.

```typescript
const base = await engine.baseAsset();
const quote = await engine.quoteAsset();

// Sell 1 wrapped base → how much quote out?
const amountOut = await engine.getQuote(base, quote, 10n ** 18n);
```

Refresh the quote close to send. Engine price is short-TTL; a stale quote often becomes `PriceExpired` or `NotEnoughAmountOut` onchain.

Suggested slippage: start at **50 bps** (`amountOutMinimum = amountOut * 9950 / 10000`). Tighten once you see fill quality.

## Approve, then swap — `NativeRouter.tradePAMM`

Do **not** call `Engine.swap()`; only the bound pool may call it. Users always go through the router.

```solidity
struct PAMMSwap {
    address pool;
    address recipient;
    address sellerToken;
    address buyerToken;
    uint256 sellerTokenAmount;
    uint256 amountOutMinimum;
    uint256 deadlineTimestamp; // unix seconds
}

function tradePAMM(PAMMSwap calldata params) external payable;
```

```typescript
import { ZeroAddress } from "ethers";

const latest = await provider.getBlock("latest");

// ERC-20: approve the router first
await (await sellerTokenContract.approve(routerAddress, sellerTokenAmount)).wait();

const tx = await router.tradePAMM(
  {
    pool: poolAddress,
    recipient,                 // where buyer tokens are sent
    sellerToken,               // address(0) if selling native
    buyerToken,                // address(0) if buying native
    sellerTokenAmount,
    amountOutMinimum,
    deadlineTimestamp: BigInt(latest.timestamp) + 1200n, // 20 minutes
  },
  { value: sellerToken === ZeroAddress ? sellerTokenAmount : 0n },
);
const receipt = await tx.wait();
```

Rules:

* Single hop only. No multi-hop routing.
* Pair must have an engine **and** `directSwapEnabled == true`.
* Sell ERC-20: `msg.value = 0`. Sell native: `sellerToken = address(0)` and `msg.value = sellerTokenAmount`.
* Buy native: `buyerToken = address(0)`; pool unwraps WETH/WBNB to `recipient`.
* `tradePAMM` returns nothing. Read actual out from the pool `PAMMTrade` event. See [events.md](events.md "mention").
* No WidgetFee on this path today.
* Failure reverts. There is no RFQ fallback if you swap directly with pAMM. Use [FirmQuote Swap APIs](../firmquote-swap-apis/ "mention") for hybrid RFQ fallback and a higher swap success rate.
