---
hidden: true
---

# WebSocket Integration

WebSocket integration serves as an alternative to HTTPS integration. While the overall flow remains the same, WebSocket allows you to stream order book data to Native instead of having Native pull the data from your server.

### Server URL & authentication

Native's WebSocket URL is hosted at: `https://newapi.native.org/v1/pmm/ws` .

To authenticate all incoming requests, you need to include the `api_key` in the headers. You can obtain an `api_key` by requesting it from the Native team, as the API key needs to be manually whitelisted for access to the WebSocket URL.

{% hint style="info" %}
**Note:** You are required to connect to our WebSocket via the `ws` library, not the `socket.IO` library. For further examples, you can visit the Market Maker WebSocket Client demo on our [GitHub](https://github.com/Native-org/market-maker-ws-client).
{% endhint %}

### **WebSocket message types**

To fully integrate with Native and utilize its full capabilities, you must implement and handle the following message types:

* [**Orderbook**](orderbook.md)**:** Enables Native to estimate prices for different order sizes.
* [**Firm-Quote**](firm-quote.md)**:** Allows Native to obtain a price from you for the requested token and amount.
* [**Sign-Quote**](sign-quote.md)**:** Enables you to validate and sign the order for on-chain execution.
