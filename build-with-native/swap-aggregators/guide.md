# Guide

Native Relay connects you with Native Core liquidity on public networks in two same-chain modes:

<table><thead><tr><th width="160">Mode</th><th>How you quote</th><th>When to use it</th></tr></thead><tbody><tr><td><strong>RFQ</strong></td><td>Offchain firm-quote API, then submit signed calldata to Native Router</td><td>Existing DEX aggregators, wallets, and solvers. Supports RFQ fallback and a higher swap success rate.</td></tr><tr><td><strong>pAMM</strong></td><td>Fully onchain <code>getQuote</code> + <code>tradePAMM</code>. No API key, no RFQ signature.</td><td>Integrators that want an AMM-style onchain path with no offchain quoting.</td></tr></tbody></table>

{% hint style="info" %}
pAMM is single-hop only and has no RFQ fallback on the direct path. Use RFQ if you need hybrid fallback. Full pAMM reference: [pamm-swap-apis](pamm-swap-apis/ "mention")
{% endhint %}

## Set up a Swap using Native Relay RFQ APIs

### 1. Get an API key

To start testing, please fill out the [form](https://forms.gle/12LaH1JQyhZqBHqw9) and our team will get back to you with the API key.

### 2. Get orderbook

The latest prices for each pair Native quotes on a specified chain are published in an orderbook.

{% hint style="info" %}
Our RFQ orderbook is refreshed several times per second from Native Core. Books that are no longer valid are dropped from the response, so always quote against the latest `/orderbook` result.
{% endhint %}

Here's how to retrieve the orderbook:

```javascript
var request = require('request');
var options = {
  'method': 'GET',
  'url': 'https://v2.api.native.org/swap-api-v2/v1/orderbook?chain=ethereum',
  'headers': {
    'apikey': 'your-api-key'
  }
};
request(options, function (error, response) {
  if (error) throw new Error(error);
  console.log(response.body);
});
```

Refer to [get-orderbook.md](firmquote-swap-apis/get-orderbook.md "mention") for the full reference.

### Sample response

```json
[
    {
        "base_symbol": "WETH",
        "quote_symbol": "USDT",
        "base_address": "0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2",
        "quote_address": "0xdac17f958d2ee523a2206206994597c13d831ec7",
        "minimum_in_base": 0,
        "side": "bid",
        "levels": [
            [
                0.0001,
                3213.12345
            ],
            [
                12.75786733219471,
                3210.15
            ],
            [
                42,
                3210.12
            ]
        ]
    }
]
```

Each level is `[base_quantity, price]` and is **not** cumulative. Prices already include your API key's fee. `minimum_in_base` is expressed in wei of the base token; requests below it fail with error `171055`.

### 3. Get a firm quote

{% hint style="warning" %}
Always pass `version=6` (the latest router version) on `/firm-quote`. If `version` is omitted, version 4 is used purely for backwards compatibility, and version 4 will be deprecated soon.
{% endhint %}

Next, obtain a quote with a price that Native is committed to. If the quote is favorable, you can execute it by submitting the associated call data to the Native Router on-chain.

```javascript
var request = require('request');
var options = {
  'method': 'GET',
  'url': 'https://v2.api.native.org/swap-api-v2/v1/firm-quote?src_chain=ethereum&from_address=0x46b2deae6eff3011008ea27ea36b7c27255ddfa9&amount=1&token_in=0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2&token_out=0xdac17f958d2ee523a2206206994597c13d831ec7&version=6',
  'headers': {
    'apikey': 'your-api-key'
  }
};
request(options, function (error, response) {
  if (error) throw new Error(error);
  console.log(response.body);
});

```

### Sample response

```json
{
  "success": true,
  "orders": [
    {
      "pool": "0x5984C239c08834dBCf80d4fd741B4Ed47fFe3D02",
      "signer": "0x26a5652812905cc994009902c4b4dff950f96775",
      "recipient": "0xbf381E1cBfdb0D02F3800010e490130D3dC73118",
      "sellerToken": "0x55d398326f99059ff775485246999027b3197955",
      "buyerToken": "0x2170ed0880ac9a755fd29b2688956bd959f933f8",
      "effectiveSellerTokenAmount": "1234567890123457000000",
      "sellerTokenAmount": "1234567890123457000000",
      "buyerTokenAmount": "672916829375120000",
      "deadlineTimestamp": 1746243550,
      "nonce": 1644529534389398715,
      "quoteId": "5250641e3fabe76a2026b456cbd34b67",
      "multiHop": false,
      "signature": "",
      "externalSwapCalldata": "",
      "amountOutMinimum": "0",
      "widgetFee": {
        "signer": "0x89e56DB8eCAe8BfC7ADC790Ba7C8B1CcB17B74e6",
        "feeRecipient": "0xfa7f2f2ce037fc87846fd118e944db50c31875eb",
        "feeRate": 0
      },
      "widgetFeeSignature": "baeac6970c8da7d2351eeee125ae155dcc1b4f977274f4eadaa3f818595b671003fa4df95d4c370ee170177fb9447cdcc74193019d842c38061fdc33ea8a4b2c1c"
    }
  ],
  "widgetFee": {
    "signer": "0x89e56DB8eCAe8BfC7ADC790Ba7C8B1CcB17B74e6",
    "feeRecipient": "0xfa7f2f2ce037fc87846fd118e944db50c31875eb",
    "feeRate": 0
  },
  "widgetFeeSignature": "",
  "recipient": "0xbf381E1cBfdb0D02F3800010e490130D3dC73118",
  "amountIn": "1234567890123457000000",
  "amountOut": "672916829375120000",
  "amountOutBeforeFee": "672916829375120000",
  "fallbackSwapDataArray": null, // deprecated, always null
  "tokenTransferFeeOnPercent": 0, // deprecated, always 0
  "txRequest": {
    "target": "0xb2d1F342D2049684Fb2f8c4eF320633415598333",
    "calldata": "0xaf7065390000000000000000000000000000000000000000000000000000000000000060000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000005984c239c08834dbcf80d4fd741b4ed47ffe3d0200000000000000000000000026a5652812905cc994009902c4b4dff950f96775000000000000000000000000bf381e1cbfdb0d02f3800010e490130d3dc7311800000000000000000000000055d398326f99059ff775485246999027b31979550000000000000000000000002170ed0880ac9a755fd29b2688956bd959f933f8000000000000000000000000000000000000000000000042ed123b0bd82372400000000000000000000000000000000000000000000000000956ae4a824942800000000000000000000000000000000000000000000000000000000068158fde00000000000000000000000000000000000000000000000016d28ae5ffeb2cbb5250641e3fabe76a2026b456cbd34b670000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000220000000000000000000000000fa7f2f2ce037fc87846fd118e944db50c31875eb0000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000024000000000000000000000000000000000000000000000000000000000000002c0000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000041baeac6970c8da7d2351eeee125ae155dcc1b4f977274f4eadaa3f818595b671003fa4df95d4c370ee170177fb9447cdcc74193019d842c38061fdc33ea8a4b2c1c000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000",
    "value": "0",
    "function": "tradeRFQT"
  },
  "source": [
    1
  ],
  "errorMessage": "",
  "quoteSource": [
    1
  ],
  "router_version": "6",
  "toWrap": false,
  "toUnwrap": false,
  "amountInOffset": 36,
  "amountOutMinimumOffset": 68
}
```

### 4. Submit call data to Native Router

To execute a firm quote, submit a transaction on-chain to call the Native Router with the returned call data.

{% hint style="info" %}
You can find all the deployed addresses with their ABIs here: [addresses.md](../../resources/addresses/README.md "mention")
{% endhint %}

Here’s an example of how to submit call data to the NativeRouter:

```typescript
import axios from 'axios';
import {ethers} from "ethers";
// import routerAbi from './nativeRouter.json'

const apiKey = 'your-api-key'
const baseUrl = 'https://v2.api.native.org/swap-api-v2/v1/';

const provider = 'https://eth.llamarpc.com';

const walletAddress = ''; // Your address
const privateKey = ''; // Your private key for the address above

// Swap input
const chain = 'ethereum'; // chain name, e.g. ethereum, bsc (not a numeric chain id)
const tokenIn = '0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE'; // ETH
const tokenOut = '0xdAC17F958D2ee523a2206206994597C13D831ec7'; // USDT
const amount = 1; // in ether, not in wei

type FirmQuoteResult = {
  orders: any[],
  widgetFee: {
    signer: string,
    feeRecipient: string,
    feeRate: number
  },
  widgetFeeSignature: string,
  txRequest: {
    target: string,
    calldata: string,
    value: string,
    function?: string
  } | null, // null when toWrap / toUnwrap is true
  toWrap: boolean,
  toUnwrap: boolean,
  amountIn: string,
  amountOut: string
}

async function callFirmQuote(): Promise<FirmQuoteResult> {
  const endpoint = 'firm-quote?';
  const headers: any = {
    apikey: apiKey, // header name is case-insensitive
  };

  const response = await axios
    .get(
      `${baseUrl}${endpoint}src_chain=${chain}&token_in=${tokenIn}&token_out=${tokenOut}&amount=${amount}&from_address=${walletAddress}&version=6`,
      {
        headers,
      }
    )
  console.log('Firm quote result', response.data)
  return response.data;
}

async function callRouterToSwap(firmQuoteResult: FirmQuoteResult) {
  const jsonRpcProvider = new ethers.JsonRpcProvider(provider);
  const signer = new ethers.Wallet(privateKey, jsonRpcProvider);
  const { txRequest } = firmQuoteResult;
  if (!txRequest) throw new Error('No tx to send (wrap/unwrap or error)');
  // Always use the target returned by the API; the router differs by `version`.
  const tx = await signer.sendTransaction({
    to: txRequest.target,
    data: txRequest.calldata,
    value: txRequest.value,
  });
  await tx.wait()
}

async function main() {
  const firmQuoteResult = await callFirmQuote();
  await callRouterToSwap(firmQuoteResult)
}

main();
```

## Set up a Swap using Native Relay pAMM

pAMM does not use the Swap API. Quote and execution are both onchain on Native Router V6. No API key and no RFQ signature.

Refer to [pamm-swap-apis](pamm-swap-apis/ "mention") for addresses, events, and reverts. Deployed V6 contracts: [addresses.md](../../resources/addresses/README.md "mention")

### 1. Confirm the pair and engine

`directSwapEnabled` is the live flag. Quotes can still work when it is false, provided the pair has an engine; swaps revert with `DirectSwapDisabled`. Always read the flag onchain before sending a swap.

On BNB Chain, QQQB–USDT is enabled for test. Other listed pairs may quote while direct swap is still disabled. Confirm yourself:

```typescript
const engine = await pool.getEngine(tokenA, tokenB); // zero = not listed
const pairKey = await pool.getPairKey(tokenA, tokenB);
const enabled = await pool.directSwapEnabled(pairKey);
const boundPool = await engine.rfqPool(); // must equal the NativeRFQPool you use
```

Also require `router.isNativePools(pool) == true`.

### 2. Get an onchain quote

Call `PropAMMEngine.getQuote`. Amounts are raw token units. For native gas token (ETH/BNB), pass the **wrapped** address — never `address(0)` on the engine.

```typescript
const base = await engine.baseAsset();
const quote = await engine.quoteAsset();

// Sell 1 wrapped base → how much quote out?
const amountOut = await engine.getQuote(base, quote, 10n ** 18n);
const amountOutMinimum = amountOut * 9950n / 10000n; // start at 50 bps
```

Refresh the quote close to send. Engine price is short-TTL.

### 3. Approve, then call `tradePAMM`

Do **not** call `Engine.swap()`; only the bound pool may call it. Users always go through the router.

Sell ERC-20: approve the router, then `msg.value = 0`. Sell native: `sellerToken = address(0)` and `msg.value = sellerTokenAmount`.

```typescript
import { ZeroAddress } from "ethers";

const latest = await provider.getBlock("latest");

await (await sellerTokenContract.approve(routerAddress, sellerTokenAmount)).wait();

const tx = await router.tradePAMM(
  {
    pool: poolAddress,
    recipient,
    sellerToken,               // address(0) if selling native
    buyerToken,                // address(0) if buying native
    sellerTokenAmount,
    amountOutMinimum,
    deadlineTimestamp: BigInt(latest.timestamp) + 1200n, // 20 minutes
  },
  { value: sellerToken === ZeroAddress ? sellerTokenAmount : 0n },
);
await tx.wait();
```

`tradePAMM` returns nothing. Read actual out from the pool `PAMMTrade` event. See [events.md](pamm-swap-apis/events.md "mention").

Staging UI (BNB Chain): [https://native-pamm-staging.vercel.app/](https://native-pamm-staging.vercel.app/)
