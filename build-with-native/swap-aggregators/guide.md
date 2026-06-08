# Guide

## About Swap API

The Swap API connects you with Native's Credit Pool liquidity and competitive quotes, powered by Native Swap Engine.

Native V2 supports two types of orders:

* [auto-sign-orders.md](../../legacy/concepts/auto-sign-orders.md "mention")
* [firm-quote.md](../../concepts/firm-quote.md "mention")

Each order type will have their own corresponding Swap API endpoints.

## Set up a Swap using FirmQuote APIs

### 1. Get an API key

To start testing, please fill out the [form](https://forms.gle/12LaH1JQyhZqBHqw9) and our team will get back to you with the API key.

### 2. Get orderbook

The latest prices for each pair Native quotes on a specified chain are published in an orderbook.

{% hint style="info" %}
Our orderbook is updated every second with the latest prices quoted by our partners
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
}
```

### 3. Get a firm quote

Next, obtain a quote with a price that Native is committed to. If the quote is favorable, you can execute it by submitting the associated call data to the Native Router on-chain.

```javascript
var request = require('request');
var options = {
  'method': 'GET',
  'url': 'https://v2.api.native.org/swap-api-v2/v1/firm-quote?chain=ethereum&address=0x46b2deae6eff3011008ea27ea36b7c27255ddfa9&amount=1&token_in=0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2&token_out=0xdac17f958d2ee523a2206206994597c13d831ec7',
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
  "fallbackSwapDataArray": null,
  "tokenTransferFeeOnPercent": 0,
  "txRequest": {
    "target": "0xb2d1F342D2049684Fb2f8c4eF320633415598333",
    "calldata": "0xaf7065390000000000000000000000000000000000000000000000000000000000000060000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000005984c239c08834dbcf80d4fd741b4ed47ffe3d0200000000000000000000000026a5652812905cc994009902c4b4dff950f96775000000000000000000000000bf381e1cbfdb0d02f3800010e490130d3dc7311800000000000000000000000055d398326f99059ff775485246999027b31979550000000000000000000000002170ed0880ac9a755fd29b2688956bd959f933f8000000000000000000000000000000000000000000000042ed123b0bd82372400000000000000000000000000000000000000000000000000956ae4a824942800000000000000000000000000000000000000000000000000000000068158fde00000000000000000000000000000000000000000000000016d28ae5ffeb2cbb5250641e3fabe76a2026b456cbd34b670000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000220000000000000000000000000fa7f2f2ce037fc87846fd118e944db50c31875eb0000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000024000000000000000000000000000000000000000000000000000000000000002c0000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000041baeac6970c8da7d2351eeee125ae155dcc1b4f977274f4eadaa3f818595b671003fa4df95d4c370ee170177fb9447cdcc74193019d842c38061fdc33ea8a4b2c1c000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000",
    "value": "0"
  },
  "source": [
    1
  ],
  "errorMessage": "",
  "router_version": "v1.0.0",
  "toWrap": false,
  "toUnwrap": false,
  "amountInOffset": 36,
  "amountOutMinimumOffset": 68
}
```

### 4. Submit call data to Native Router

To execute a firm quote, submit a transaction on-chain to call the Native Router with the returned call data.

{% hint style="info" %}
You can find all the deployed addresses with their ABIs here: [addresses.md](../../resources/addresses.md "mention")
{% endhint %}

Here’s an example of how to submit call data to the NativeRouter:

```typescript
import axios from 'axios';
import {ethers} from "ethers";
// import routerAbi from './nativeRouter.json'

const apiKey = 'your-api-key'
const baseUrl = 'https://v2.api.native.org/swap-api-v2/v1/';

const provider = 'https://eth.llamarpc.com';
const routerAddress = '0x5C0aBf0F651613696A5c57efafC6ab59A460B32d'; // Refer to Contract Address section -> NativeRouter (Proxy) for the address

const walletAddress = ''; // Your address
const privateKey = ''; // Your private key for the address above

// Swap input
const chain = 'ethereum'; // 1, 56
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
  calldata: string,
  amountIn: string,
  amountOut: string,
  fallbackData: string
}

async function callFirmQuote(): Promise<FirmQuoteResult> {
  const endpoint = 'firm-quote?';
  const headers: any = {
    api_key: apiKey,
  };

  const response = await axios
    .get(
      `${baseUrl}${endpoint}chain=${chain}&token_in=${tokenIn}&token_out=${tokenOut}&amount=${amount}&address=${walletAddress}`,
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
  const tx = await signer.sendTransaction({to: routerAddress, data: firmQuoteResult.calldata});
  await tx.wait()
}

async function main() {
  const firmQuoteResult = await callFirmQuote();
  await callRouterToSwap(firmQuoteResult)
}

main();
```
